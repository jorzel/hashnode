## Different types of application logic

Every project starts with functional requirements. We just want to have an application that does what we have specified. In this post I would like to show that business rules can fall into various categories and what implications it could have.

Let's take an example. Suppose, we have a system with customers and there is a feature that each customer can send vouchers to friends. When user activates a voucher, she becomes a customer of the system with some starting bonus points. We have also some rules specified that clarify the process:
- voucher can be sent only by system users (customers)
- voucher is sent by passing a target email address, however there is only several email domains allowed
- each voucher has bonus points. The value is dependent on user status. VIP customer can send voucher with 10 bonus points, while regular user with 3 points
- when process is completed, voucher should be sent to given email address
- each customer can only send maximum 3 vouchers to different emails
- customer can list sent vouchers. He can also see which vouchers have been activated

*Following implementations would be simplified for the sake of readability. We assume that our models are properly mapped to ORM classes or database tables and persisted, but it would be omitted in code listings.

## Straightforward approach
Our system would need at least two models: `Voucher(customer_id: int, points: int, is_active: bool, email: str)` represents a voucher sent by a customer and `Customer(id: int, is_vip: bool)` represents the customer that can send the voucher. 
We can start with straightforward approach and implement all functional requirements inside two HTTP controllers (for code implementation, we use Python Flask framework + SQLAlchemy ORM, however it should be clear also for non-pythonic readers). The first one would be an endpoint that make POST request to send a voucher to given email:
```python
AVAILABLE_DOMAINS = ["test.com", "db.test"]
MAX_VOUCHERS = 3

def get_customer(request: Request, session: Session) -> int:
    customer_id = # retrieve customer_id from request token to authenticate
    return session.query(Customer).get(customer_id)

def send_voucher_mail(email: str) -> None:
    # implement email sender to send message with a voucher

@app.route("/vouchers", methods=["POST"])
def send_voucher(self):
    current_customer = get_customer(request)
    if not current_customer:
        raise AuthException()
    email = request.get_json().get("email")
    if not email:
        raise InputDataException()
    if email.split("@")[1] not in AVAILABLE_DOMAINS:
        raise InvalidEmailDomain()
    if (
        session.query(Voucher).filter_by(
            customer_id=current_customer.id
        ) >= MAX_VOUCHERS
    ):
        raise MaxVoucherException()
    if (
        session.query(Voucher)
        .filter_by(
            customer_id=current_customer.id, email=email
         )
        .first()
    ):
        raise VoucherAlreadySent()
    voucher_points = 10 if current_customer.is_vip else 3
    voucher = Voucher(
        points=voucher_points, 
        customer_id=current_customer.id
    )
    session.add(voucher)
    session.commit()
    send_voucher_mail(email)
    return "Success"
```
The other endpoint lists all vouchers that the customer has sent using GET request:
```python

@app.route("/vouchers", methods=["GET"])
def vouchers(self):
    current_customer = get_customer(request)
    if not current_customer:
        raise AuthException()
    is_active = bool(request.args.get("is_active", 0))
    vouchers = session.query(Voucher).filter_by(
        customer_id=current_customer.id
    )
    if is_active:
        vouchers = vouchers.filter_by(is_active=is_active)
    return jsonify(
        [{"is_active": v.is_active, "points": v.points} for v in vouchers]
    )
```
That solution would work and is good enough for fast prototyping. However, when our system is getting larger, it can lead to some problems. We have all requirements implemented in the same manner, while each of them can be categorized as different logic type:
- authentication and authorization
- domain logic (invariants)
- process logic (orchestration, coordination)
- calculations (algorithms)
- presentation logic
- validation 

Now, we will try to fit our requirements into above categories and see what profits it can bring to us.

## Separation of concerns
### Authentication and authorization
Authentication step determine user identity (whether user is signed in or 'recognized' by a system), while authorization refers to whether an authenticated user has access to a given resource. In our code, we have only authentication step that can be enforced at HTTP controller entry. We can use decorator that guard access to the endpoint or take `get_customer` function that would be executed at the very beginning of the endpoint.

```python
def get_customer(request: Request, session: Session) -> int:
    customer_id = # retrieve customer_id from request token to authenticate
    return session.query(Customer).get(customer_id)

@app.route("/vouchers", methods=["POST"])
def send_voucher(self):
    current_customer = get_customer(request)
    if not current_customer:
        raise AuthException()
    ...
```

### Validation
This type of logic is focused on incoming data and check its correctness. It does not need any knowledge about current system state, so can be executed in HTTP controller (as soon as possible). In our case, the validation step would be checking whether email address is proper and email domain belongs to given list. We can represent our incoming email address as a value object `Email` that has parsed `domain` property.
```python
AVAILABLE_DOMAINS = ["test.com", "db.test"]

class Email:
    def __init__(self, email_address: str):
        if "@" not in email_address:
            raise EmailValidationError()
        self.address = email_address

    @property
    def domain(self):
        return self.address.split("@")[1]

@app.route("/vouchers", methods=["POST"])
def send_voucher(self):
    ...
    if not request.get_json().get('email'):
        raise InputDataException()
    email = Email(request.get_json()["email"])
    if email.domain not in AVAILABLE_DOMAINS:
        raise InvalidEmailDomain()
    ...
```

### Domain logic (invariants)
It is a core of business logic. Here we have decisions that are based on current state of the system (need to query database to get information). The best place to implement it is a domain layer. In our example, it would be the step checking whether a customer have not already sent a voucher to given email and also that the customer have not exceed the number of available vouchers. We can encapsulate these logic within DDD-style aggregate `VoucherSender` that would only protect our invariants.

```python
MAX_VOUCHERS = 3

class VoucherSender:
    customer_id: int
    emails: list[str]

    def _check_send(self, email: Email) -> None:
        if email.address in self.emails:
            raise VoucherAlreadySent()
        if len(self.emails) >= MAX_VOUCHERS:
            raise MaxVoucherException()

    def register(self, email: Email, points: int) -> Voucher:
        self._check_send(email)
        self.emails.append(email.address)
        return Voucher(points=points, customer_id=self.customer_id, email=email.address)
```

### Calculation
Separation of calculation can form a kind of black box. In our case, we just want to have `points` value for our voucher and to do it we use standalone function that calculates it.
```python
def calculate_points(is_vip: bool) -> int:
    return 10 if is_vip else 3

```
In the future, we can refine the implementation and use machine learning algorithm or customer-dependent policy pattern leaving the rest of our code intact. 

### Process logic
Process or coordination logic can show what your application really do (but not how). In this orchestration part of a code we define the sequence of actions and delegate execution to other classes / functions (business process skeleton). This type of logic is usually placed in a service layer and is also responsible for retrieving objects from database and transaction management.

```python
def send_voucher_mail(email: str) -> None:
    # implement email sender to send message with a voucher

def send_voucher(session: Session, customer: Customer, email: Email) -> None:
    with transaction_scope(session):
        voucher_sender = get_or_create(
            session, VoucherSender, customer_id=customer_id
        ) 
        voucher_sender.register(email, calculate_points(customer.is_vip))
    send_voucher_mail(email)

@app.route("/vouchers", methods=["POST"])
def send_voucher(self):
    ...
    send_voucher(session, current_customer, email)
    return "Success"
```
We assume that our toolkit provides `get_or_create` functionality and we have a context manager `transaction_scope` to manage database transaction (`commit` / `rollback`).,

### Presentation logic
Presentation logic often exploit read models to show something in User Interface. Depending on input parameters, set of results are filtered and some attributes are hidden or transformed. This step in our application is represented by `vouchers` GET endpoint. HTTP controller would parse incoming parameters, use `get_vouchers` function that performs filtering and serialize results.

```python
def get_vouchers(customer_id: int, **filters):
    vouchers = session.query(Voucher).filter_by(customer_id=customer_id)
    if filters:
        vouchers = vouchers.filter_by(**filters)
    return vouchers

@app.route("/vouchers", methods=["GET"])
def vouchers(self):
    ...
    is_active = bool(request.args.get("is_active", 0))
    vouchers = get_vouchers(current_customer.id, is_active=is_active)
    return jsonify(
         [{"is_active": v.is_active, "points": v.points} for v in vouchers]
     )
```

## Conclusion 
At low code level rules usually are represented by sequences of "ifs" or loops. The most important thing here is to be aware that not all functional or business requirements have the same character.  Above categorization can improve separation of concerns in large systems by creating conditions to choose a pattern that fits best to given category and place it in a proper layer. It can also make our code more testable. On the one hand, we can have comprehensive domain logic tests, and provide only happy path check for process logic.
You don't have to introduce this separation from day one. Even if you attribute categories to your requirements items, it could improve your understanding of business demands massively.

