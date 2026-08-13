I am learning the Frappe Payments app as a backend developer.

I already understand the basic Frappe concepts, including:

* DocType
* Document
* Single DocType
* Controller
* lifecycle methods like `on_update`
* whitelisted methods
* Integration Request
* hooks
* how a DocType JSON file relates to its Python controller

Now I want to understand the **Payments module deeply before I continue modifying the Chapa integration**.

Please act like a senior Frappe backend developer teaching a junior backend developer.

## Main goal

Explain the Payments module to me **file by file, function by function, and code block by code block**.

Do not give me only a high-level summary.

I want to understand what every important piece of code does, why it exists, how it connects to other files, and what happens at runtime.

Assume I know Python basics, but I am still new to Frappe.

## Important teaching style

Explain things in a very detailed but beginner-friendly way.

For each important piece of code, use this structure:

### 1. Show the code

Example:

```python
def get_payment_gateway_controller(payment_gateway):
    ...
```

### 2. Explain every part

Explain:

* what `def` means in this specific context
* what each parameter contains
* where the function is called from
* what it returns
* why the return value is needed
* which Frappe APIs are being used
* what database access happens
* what object type is being returned
* what happens next after the function returns

### 3. Explain the runtime flow

Show something like:

```text
Caller
   ↓
get_payment_gateway_controller("Chapa")
   ↓
Payment Gateway record
   ↓
gateway_controller
   ↓
frappe.get_doc(...)
   ↓
Chapa Settings Document
   ↓
returned to caller
```

### 4. Give a real example

For example, if the input is:

```python
get_payment_gateway_controller("Chapa")
```

show what values may exist during execution:

```text
payment_gateway = "Chapa"

Payment Gateway record:
gateway = "Chapa"
gateway_settings = "Chapa Settings"
gateway_controller = "Chapa Settings"
```

Then explain what the final returned object looks like conceptually.

## Very important: explain syntax too

Whenever you see syntax that may be confusing to a beginner, explain it.

Examples:

```python
**kwargs
```

Explain how keyword arguments work.

```python
frappe._dict(kwargs)
```

Explain why Frappe uses `_dict` instead of a normal dict.

```python
self.data.amount
```

Explain where `self.data` came from and why dot notation works.

```python
result["data"]["checkout_url"]
```

Explain the nested dictionary access.

```python
@frappe.whitelist(allow_guest=True)
```

Explain decorators and what Frappe does with this decorator.

```python
from frappe import _
```

Explain translation functions.

```python
f"Bearer {secret_key}"
```

Explain the f-string and the resulting string.

```python
("ETB",)
```

Explain why the trailing comma matters.

Do this whenever similar syntax appears.

## Order I want you to follow

Please inspect the actual repository and explain the Payments module in this order.

### PART 1 — Payment Gateway foundation

Start with the Payment Gateway DocType.

Explain:

* `payments/payments/doctype/payment_gateway/payment_gateway.json`
* `payments/payments/doctype/payment_gateway/payment_gateway.py`

Explain every important field in the JSON.

Especially explain:

* `gateway`
* `gateway_settings`
* `gateway_controller`

Then explain why the Python controller might only contain:

```python
class PaymentGateway(Document):
    pass
```

Explain what `pass` means and what functionality is still inherited from `Document`.

Do not continue until this part is fully explained.

---

### PART 2 — Generic payment utility functions

Then explain:

```text
payments/utils/utils.py
```

Focus especially on:

```python
get_payment_gateway_controller()
```

and:

```python
create_payment_gateway()
```

For each one, explain:

* every line
* database queries
* `frappe.db.exists`
* `frappe.get_doc`
* `.insert()`
* object types
* return values
* what happens if the gateway exists
* what happens if it does not exist

Then connect the two functions together.

Show the flow:

```text
Chapa Settings saved
        ↓
create_payment_gateway(...)
        ↓
Payment Gateway created
        ↓

Later:

Payment code receives "Chapa"
        ↓
get_payment_gateway_controller("Chapa")
        ↓
Chapa Settings Document returned
```

---

### PART 3 — Web Form payment flow

Then explain:

```text
payments/overrides/payment_webform.py
```

Explain the complete payment flow.

I especially want to understand this dictionary:

```python
payment_details = {
    "amount": amount,
    "title": title,
    "description": title,
    "reference_doctype": doc.doctype,
    "reference_docname": doc.name,
    "payer_email": frappe.session.user,
    "payer_name": frappe.utils.get_fullname(frappe.session.user),
    "currency": self.currency,
    "redirect_to": ...
}
```

Explain every field individually.

For each field, tell me:

* where the value comes from
* an example value
* why the payment gateway needs it

For example:

```text
reference_doctype = "Event Registration"
reference_docname = "EVREG-0001"
```

Explain how these values connect a Chapa transaction back to a Frappe document.

Then explain:

```python
controller.get_payment_url(**payment_details)
```

in extreme detail.

Show how:

```python
payment_details
```

becomes:

```python
get_payment_url(
    amount=...,
    title=...,
    reference_doctype=...,
    ...
)
```

---

### PART 4 — Chapa Settings JSON

Then explain:

```text
payments/payment_gateways/doctype/chapa_settings/chapa_settings.json
```

Explain each field.

For every field tell me:

* fieldname
* fieldtype
* whether it is required
* where the Python code uses it
* whether it currently appears unused

Especially explain:

```json
{
    "fieldname": "secret_key",
    "fieldtype": "Password"
}
```

Explain:

```python
self.get_password("secret_key")
```

and why using:

```python
self.secret_key
```

may be wrong for an API request.

---

### PART 5 — ChapaSettings Python controller

Then explain the main file:

```text
payments/payment_gateways/doctype/chapa_settings/chapa_settings.py
```

This is the most important part.

Go through the file from top to bottom.

Do not skip code.

Explain imports first.

For example:

```python
import uuid
import re
import requests
import frappe

from frappe import _
from frappe.model.document import Document
```

For every import, explain:

* what library/module it is
* whether it is Python standard library, external package, or Frappe
* exactly where it is used in this file

Then explain the class:

```python
class ChapaSettings(Document):
```

Then each method separately.

For example:

```python
def on_update(self):
```

Explain:

* when Frappe calls it
* what `self` contains
* why `self.name` is important
* why `create_payment_gateway()` is called here

Then:

```python
validate_credentials()
```

Explain the entire HTTP request.

Show the final request conceptually:

```text
GET https://api.chapa.co/v1/banks

Authorization:
Bearer CHASECK-xxxxxxxx
```

Explain:

```python
timeout=10
```

and error handling.

Then explain:

```python
validate_transaction_currency()
```

including:

```python
supported_currencies = ("ETB",)
```

Then explain:

```python
get_payment_url(self, **kwargs)
```

in detail.

Show what `kwargs` could contain at runtime.

Then explain:

```python
self.data = frappe._dict(kwargs)
```

and show:

```python
self.data.amount
self.data.currency
self.data.reference_docname
```

Then explain transaction initialization.

---

### PART 6 — tx_ref

Spend extra time explaining:

```python
tx_ref = f"{self.data.reference_docname}-{uuid.uuid4().hex[:10]}"
```

Break it into parts:

```python
self.data.reference_docname
```

```python
uuid.uuid4()
```

```python
.hex
```

```python
[:10]
```

and the f-string.

Show an actual example result.

Also explain why a payment system needs a unique transaction reference.

Explain what could go wrong if two payment attempts reused the same `tx_ref`.

---

### PART 7 — Chapa payload

Explain the complete payload sent to Chapa.

For example:

```python
payload = {
    "amount": str(self.data.amount),
    "currency": "ETB",
    "email": email,
    "first_name": ...,
    "tx_ref": tx_ref,
    "callback_url": ...,
    "return_url": ...,
    "customization": {...},
}
```

Explain every field.

Very importantly explain the difference between:

```text
callback_url
```

and:

```text
return_url
```

Use diagrams.

For example:

```text
callback_url

Chapa Server
      ↓
Your Frappe Backend


return_url

Customer Browser
      ↓
Your Website
```

Explain why `return_url` must never be trusted as proof that payment succeeded.

---

### PART 8 — requests.post()

Explain this code deeply:

```python
response = requests.post(
    url,
    json=payload,
    headers=headers,
    timeout=30,
)
```

Explain:

* HTTP POST
* URL
* JSON body
* headers
* bearer token
* timeout

Then:

```python
response.raise_for_status()
```

Explain exactly what it does.

Then:

```python
result = response.json()
```

Explain the resulting Python object.

Then:

```python
result["data"]["checkout_url"]
```

Explain nested dictionary access.

Show an example Chapa response and demonstrate how Python reaches the checkout URL.

---

### PART 9 — callback / verification

Then explain:

```python
@frappe.whitelist(allow_guest=True)
def verify_payment():
```

Explain decorators first.

Explain:

* why it must be whitelisted
* why Chapa needs `allow_guest=True`
* what security risk that introduces
* why `allow_guest=True` does not mean we should trust the request

Then explain:

```text
/api/method/...
```

and how Frappe maps the URL to the Python function.

Go line-by-line through the callback.

Explain how it gets transaction references from the request.

Then explain the verification API call:

```text
GET /transaction/verify/<reference>
```

Explain why server-side verification is required.

---

### PART 10 — explain problems separately

After I understand how the existing callback works, explain its problems one by one.

Do not mix the explanation of the current code with the solution at first.

For each problem, use:

```text
Current code
↓
What it assumes
↓
Why that assumption is dangerous/wrong
↓
Possible failure scenario
↓
Correct design
```

Cover at least:

* `tx_ref` vs `trx_ref`
* hard-coded LMS Payment
* no locally stored expected transaction
* no amount verification
* no currency verification
* no reference verification
* webhook secret unused
* no idempotency
* direct LMS-specific update
* missing `on_payment_authorized()`
* hard-coded fallback email
* hard-coded description
* sensitive debugging
* exposing traceback
* manual `frappe.db.commit()`

---

### PART 11 — Integration Request

Then teach me Frappe's Integration Request in detail.

Do not assume I know it.

Explain:

* what DocType it is
* what problem it solves
* when a record should be created
* what should be stored before redirecting to Chapa
* how it should be found during verification
* how status changes

Use a complete example:

```text
Integration Request

tx_ref = EVENT-REG-0001-a8213cd92f
amount = 500
currency = ETB
reference_doctype = Event Registration
reference_docname = EVREG-0001
status = Queued
```

Then show what happens after successful Chapa verification.

---

### PART 12 — reference document and on_payment_authorized

Explain:

```python
reference_doc.run_method(
    "on_payment_authorized",
    "Completed",
)
```

Explain:

* what `reference_doc` is
* how Frappe retrieves it
* what `run_method()` does
* why the payment gateway should not directly know how Event Registration, Ticket Order, Sales Invoice, etc. marks itself paid

Use an example like:

```python
class EventRegistration(Document):

    def on_payment_authorized(self, status):
        if status == "Completed" and not self.payment_received:
            self.db_set("payment_received", 1)
```

Explain every line.

---

### PART 13 — idempotency

Teach idempotency carefully.

Assume I have never learned this concept.

Use an example where Chapa sends the same webhook three times.

Show the dangerous behavior:

```text
Webhook 1 → create ticket
Webhook 2 → create another ticket
Webhook 3 → create another ticket
```

Then show the correct behavior:

```text
Webhook 1 → complete transaction
Webhook 2 → transaction already completed → do nothing
Webhook 3 → transaction already completed → do nothing
```

Explain how Integration Request can help achieve this.

---

### PART 14 — duplicate files

Inspect and explain these too:

```text
chapa_integration.py
chapa.py
chapa_checkout.py
chapa_checkout.html
chapa_settings.js
test_chapa_settings.py
```

For each one explain:

* what the file currently does
* whether anything imports or calls it
* whether it participates in the current payment flow
* whether it duplicates another implementation
* whether it should probably be kept, refactored, or removed

Do not delete or modify anything yet.

I only want understanding.

---

## Critical instruction

Do NOT modify any code.

Do NOT refactor anything yet.

Do NOT implement fixes yet.

Do NOT jump straight to a proposed final architecture.

First teach me exactly how the current codebase works.

Whenever you say:

> "this function calls X"

show me where X is defined if it is inside this repository.

Whenever you say:

> "this method is called automatically"

explain what Frappe event causes it.

Whenever a value enters a function, tell me where that value came from.

Whenever a function returns a value, tell me who consumes it next.

I want to be able to follow the data from:

```text
User clicks Pay
```

all the way through:

```text
Frappe
→ Payment Gateway
→ Chapa Settings
→ Chapa API
→ checkout URL
→ customer payment
→ callback
→ verification
→ reference document
```

At the end of each major part, give me:

1. a short recap,
2. a runtime flow diagram,
3. 3–5 questions to test whether I understood it.

Most importantly: **do not rush through the code**. Explain each important line like a mentor sitting next to me and reading the repository with me.

Start only with **PART 1 — Payment Gateway foundation**. After fully explaining it, stop there so I can study it before moving to PART 2.
