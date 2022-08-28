# Flask-PaycomUz
> Only for Uzbekistan

Paycom.uz integration for Flask

## Links
* [About PaycomUz](https://https://business.payme.uz/)
* [PaycomUz Docs](https://developer.help.paycom.uz/ru)
* [PaycomUz Sandbox](https://test.paycom.uz/instruction)
## How it Works


### Install 

```
pip install Flask-PaycomUz
```

### Add your credentials from PaycomUz to config file

```python
PAYCOM_ID = "your merchant ID"
PAYCOM_KEY = "your sercet key" 
```

### Create Flask App With Flask-PaycomUz


```python
from flask_paycomuz import Paycom
from flask import *
from flask_sqlalchemy import SQLAlchemy

db = SQLAlchemy()
paycom = Paycom(db)

def CheckAllowment(account): # Before creating transaction Flask-PaycomUz send account data which will have key whick you gave to Register_Account_data to validate it. Return True/False to Validate
    order_id = account.get('order_id') 
    res = Order.query.filter(Order.id == order_id).first()
    if res:
        return True
    return False 


def create_app():
    app = Flask(__name__)
    app.config.from_pyfile('config.py')
    
    paycom.Register_Account_Data(['order_id']) # Register Paycom payment details, requires to set before init app
    paycom.Register_Validators(CheckAllowment) # Register your Validator for payment, requires to set before init app
    paycom.init_app(app, url_prefix='/payme') # Init Paycom to your application, url_prefix to add view to app, defualt /payme, in that case Flask-PaycomUz register JSON-RPC route to recieve requests from PaycomUz as https://example.uz/payme
    
    @app.get("/")
    def index():
        return "Hello World"
    

    @app.get('/create_transaction')
    def create_transaction():
        amount = 10000
        order_id = 45 # Order ID to create Transaction in Paycom
        return_url = "https://example.uz/return_url" # return Url after successful or error payment 
        url = paycom.Generate(amount=amount, return_url=return_url, order_id = order_id)

    with app.app_context():
        db.create_all() # Don't forget to create db, flask_paycomuz adds 2 table to db, Payme_Transaction and Payme_Account


    return app

app = create_app()

if __name__ == "__main__":
    app.run(debug=True)
```

### Flask-PaycomUz models schema


```python
"""
Flask-PaycomUz uses Flask-SQLAlchemy models to save data in database, it prefers to use Postgresql
"""
class Payme_Transaction(db.Model):
            __tablename__ = 'payme_transaction'
            id = db.Column(db.Integer, primary_key = True)
            payme_id = db.Column(db.Integer, unique=True)
            transaction_id = db.Column(db.String, unique=True)
            time = db.Column(db.BigInteger, nullable=True)
            amount = db.Column(db.Integer, nullable=True)
            state = db.Column(db.Integer, default=1)
            created_at = db.Column(db.BigInteger, nullable=False)
            account = db.relationship('Payme_Account', backref='payme_transaction')

            def result(self):
                return {"result" : {
                            "create_time" : self.created_at,
                            "transaction" : self.transaction_id,
                            "state" : self.state}
                    }
        
        class Payme_Account(db.Model):
            __tablename__ = 'payme_account'
            id = db.Column(db.Integer, primary_key = True)
            transaction_id = db.Column(db.ForeignKey("payme_transaction.transaction_id"))
            key = db.Column(db.String, nullable=True)
            value = db.Column(db.String, nullable=True)
```
### Accessing Models


Select data from models

```python
paycom.models # it contains 2 models in list, [Payme_Transaction,Payme_Account ]
Payme_Transaction, Payme_Account = paycom.models
transaction = Payme_Transaction.query.all() # You can select or filter data
```

Add model view to Flask-Admin

```python
from flask_admin.contrib.sqla import ModelView
admin.add_view(ModelView(paycom.models[0], db.session))
admin.add_view(ModelView(paycom.models[1], db.session))

```

## Task List

### Merchant API methods

- [x] CheckPerformTransaction
- [x] CreateTransaction
- [x] CheckTransaction
- [x] PerformTransaction
- [ ] CancelTransaction

## Licence
This project is licensed under the MIT License (see the `LICENSE` file for details).