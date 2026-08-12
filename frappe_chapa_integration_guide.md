You already have a partial Chapa integration in this repository, but it is incomplete and unsafe for production. It can create a Chapa checkout URL, but its callback will likely fail and payment verification is not strict enough.

I did not change any code yet. First, here is how the module works.

## 1. Basic Frappe concepts

A Frappe app mainly uses:

- **DocType**: similar to a database model plus form definition.
- **Document**: a Python object representing one database record.
- **Single DocType**: one settings record for the entire site.
- **Controller**: the Python class containing a DocType’s behavior.
- **Whitelisted method**: a Python function accessible through `/api/method/...`.
- **Integration Request**: Frappe’s standard transaction/API log.
- **Hook**: code that Frappe runs on events such as installation or document saving.

For example:

```
class ChapaSettings(Document):
    def on_update(self):
        ...
```

This means:

- `ChapaSettings` controls the Chapa Settings DocType.
- `Document` gives it methods such as `save()`, `insert()` and `get_password()`.
- `on_update()` runs automatically after the settings are saved.

## 2. Overall payment flow

The intended flow is:

```
Customer submits a payment form
        ↓
Frappe finds the selected Payment Gateway
        ↓
ChapaSettings.get_payment_url()
        ↓
POST /transaction/initialize to Chapa
        ↓
Chapa returns checkout_url
        ↓
Customer pays on Chapa
        ↓
Chapa calls your callback/webhook
        ↓
Your server calls Chapa's verify endpoint
        ↓
Validate status, amount, currency and reference
        ↓
Mark the local document as paid
```

The browser redirect must never be enough to mark a payment as successful. Your server must verify it directly with Chapa.

## 3. The generic Payments code

### Payment Gateway DocType

[Payment Gateway]\(/home/anu/frappe/frappe-bench/apps/payments/payments/payments/doctype/payment\_gateway/payment\_gateway.json) is a registry of available payment providers.

Its important fields are:

- `gateway`: displayed name, such as `Chapa`.
- `gateway_settings`: settings DocType, such as `Chapa Settings`.
- `gateway_controller`: the particular settings record to use.

The Python controller currently contains only:

```
class PaymentGateway(Document):
    pass
```

`pass` means “there is no additional Python behavior.” The standard behavior inherited from `Document` is enough.

### Creating a gateway

[utils.py (line 48)]\(/home/anu/frappe/frappe-bench/apps/payments/payments/utils/utils.py:48) contains:

```
def create_payment_gateway(gateway, settings=None, controller=None):
```

It checks whether a Payment Gateway exists. If not, it inserts one:

```
{
    "doctype": "Payment Gateway",
    "gateway": gateway,
    "gateway_settings": settings,
    "gateway_controller": controller,
}
```

For Chapa, that becomes approximately:

```
Gateway: Chapa
Settings DocType: Chapa Settings
Controller record: Chapa Settings
```

### Finding the controller

[get\_payment\_gateway\_controller() (line 14)]\(/home/anu/frappe/frappe-bench/apps/payments/payments/utils/utils.py:14) receives `"Chapa"` and returns the Chapa Settings document.

That allows generic payment code to do:

```
controller = get_payment_gateway_controller("Chapa")
controller.get_payment_url(...)
```

The caller does not need to know how Chapa itself works.

### Web Form payment flow

[payment\_webform.py (line 25)]\(/home/anu/frappe/frappe-bench/apps/payments/payments/overrides/payment\_webform.py:25) integrates payments with Frappe Web Forms.

It builds this dictionary:

```
payment_details = {
    "amount": amount,
    "title": title,
    "description": title,
    "reference_doctype": doc.doctype,
    "reference_docname": doc.name,
    "payer_email": frappe.session.user,
    "payer_name": frappe.utils.get_fullname(frappe.session.user),
    "currency": self.currency,
    "redirect_to": ...,
}
```

Important meanings:

- `reference_doctype`: type of record being paid for.
- `reference_docname`: exact record name.
- `amount`: expected payment amount.
- `currency`: expected currency.
- `redirect_to`: where the browser should go afterward.

It then calls:

```
controller.get_payment_url(**payment_details)
```

The `**` expands a dictionary into named arguments.

## 4. Chapa Settings DocType

[chapa\_settings.json]\(/home/anu/frappe/frappe-bench/apps/payments/payments/payment\_gateways/doctype/chapa\_settings/chapa\_settings.json) defines the settings form.

Currently it contains:

- Gateway Name
- Merchant ID
- Public Key
- Secret Key
- Webhook Secret
- Use Sandbox
- Header Image

The important field is:

```
{
  "fieldname": "secret_key",
  "fieldtype": "Password"
}
```

Frappe encrypts Password fields. Python retrieves the real value with:

```
self.get_password("secret_key")
```

Do not use `self.secret_key` for an API call because that can contain a masked/encrypted representation.

Several fields are currently unused:

- `merchant_id`
- `public_key`
- `webhook_secret`
- `use_sandbox`
- `header_img`

In particular, making unused fields mandatory will make configuration unnecessarily difficult.

## 5. ChapaSettings Python controller

The main implementation is [chapa\_settings.py]\(/home/anu/frappe/frappe-bench/apps/payments/payments/payment\_gateways/doctype/chapa\_settings/chapa\_settings.py).

### Imports

```
import uuid
import re
import requests
import frappe
```

- `uuid`: creates random values for unique transaction references.
- `re`: cleans text using regular expressions.
- `requests`: sends HTTP requests.
- `frappe`: provides database, settings, exceptions and API features.

```
from frappe import _
```

`_()` marks text as translatable:

```
_("Invalid Chapa Secret Key")
```

```
from frappe.model.document import Document
```

This provides the base class for the settings controller.

### Supported currencies

```
supported_currencies = ("ETB",)
```

A one-item Python tuple must have a trailing comma.

The current Chapa documentation lists ETB and USD for hosted checkout, but restricting this project to ETB may be an intentional business decision. [Chapa payment initialization documentation](https://developer.chapa.co/integrations/accept-payments)

This declaration is duplicated at lines 17 and 56. One should be removed.

### Registering Chapa

```
def on_update(self):
    create_payment_gateway(
        "Chapa",
        settings=self.name,
        controller=self.name,
    )

    self.validate_credentials()
```

When a System Manager saves Chapa Settings:

1. It creates the `Chapa` Payment Gateway if missing.
2. It checks the secret key.

`self.name` is `"Chapa Settings"` because this is a Single DocType.

### Validating credentials

```
headers = {
    "Authorization": f"Bearer {self.get_password('secret_key')}"
}
```

An f-string inserts the secret into the header.

The code calls:

```
requests.get(
    "https://api.chapa.co/v1/banks",
    headers=headers,
    timeout=10,
)
```

If Chapa does not return HTTP 200, it throws:

```
frappe.throw(_("Invalid Chapa Secret Key"))
```

`frappe.throw()` stops execution and shows a validation error.

### Validating currency

```
def validate_transaction_currency(self, currency):
    if currency and currency not in self.supported_currencies:
        frappe.throw(_("Chapa only supports ETB transactions"))
```

This prevents sending an unsupported currency.

### Receiving payment details

```
def get_payment_url(self, **kwargs):
    self.validate_transaction_currency(kwargs.get("currency"))
    self.data = frappe._dict(kwargs)
    return self.initialize_transaction()
```

`kwargs` is a normal dictionary containing all named arguments.

`frappe._dict` behaves like a dictionary but also permits dot access:

```
self.data.amount
self.data.reference_docname
```

instead of:

```
self.data["amount"]
self.data["reference_docname"]
```

### Creating `tx_ref`

```
tx_ref = f"{self.data.reference_docname}-{uuid.uuid4().hex[:10]}"
```

Example:

```
LMS-PAY-0005-8e42f37ac1
```

Every payment attempt needs its own unique reference. Reusing a reference can mix two payment attempts.

### Building the Chapa payload

```
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

Important distinction:

- `callback_url`: Chapa calls this server endpoint.
- `return_url`: Chapa redirects the customer’s browser here.

Chapa requires hosted checkout initialization through `POST /transaction/initialize`, using the secret key as a bearer token. [Chapa acceptance documentation](https://developer.chapa.co/integrations/accept-payments)

### Calling Chapa

```
response = requests.post(
    url,
    json=payload,
    headers=headers,
    timeout=30,
)
```

- `POST` sends data to create something.
- `json=payload` converts the dictionary into JSON.
- `timeout=30` prevents the worker from waiting forever.

```
response.raise_for_status()
```

This raises an exception for HTTP errors such as 400, 401 or 500.

```
result = response.json()
```

This converts Chapa’s JSON response into a Python dictionary.

```
return result["data"]["checkout_url"]
```

The URL is returned to the browser, which redirects the customer to Chapa.

## 6. Current callback

The endpoint is:

```
@frappe.whitelist(allow_guest=True)
def verify_payment():
```

`allow_guest=True` is necessary because Chapa is not logged into your Frappe site.

Its URL becomes:

```
/api/method/payments.payment_gateways.doctype.chapa_settings.chapa_settings.verify_payment
```

The function obtains a reference, calls:

```
GET https://api.chapa.co/v1/transaction/verify/<reference>
```

and then marks a local LMS Payment as paid.

The use of the verification endpoint is correct in principle. Chapa explicitly requires server-side verification. [Chapa verification documentation](https://developer.chapa.co/integrations/verify-payments)

## 7. Serious problems in the current callback

These must be fixed before accepting real money.

1. **Wrong callback parameter**

The code reads:

```
tx_ref = frappe.form_dict.get("tx_ref")
```

Chapa’s documented callback response uses `trx_ref`, not `tx_ref`. Therefore the current callback may always throw `Missing tx_ref`.

2. **It only supports LMS Payment**

```
if frappe.db.exists("LMS Payment", payment_name):
```

There is no LMS app or LMS Payment DocType in this bench. The installed code contains only Frappe, Payments and the almost-empty Event Management app.

So successful payments currently cannot update a relevant local record.

3. **No expected transaction record**

The system does not save the expected:

- `tx_ref`
- amount
- currency
- reference DocType
- reference document
- payment status

before sending the customer to Chapa.

A callback therefore has no reliable local transaction against which to compare Chapa’s result.

4. **Amount and currency are not compared**

A safe callback must confirm:

```
verified amount == expected amount
verified currency == expected currency
verified tx_ref == expected tx_ref
verified transaction status == success
```

Checking only:

```
result.get("status") == "success"
```

is not sufficient.

5. **Webhook secret is unused**

The settings include `webhook_secret`, but the code never verifies it.

Chapa sends signed webhooks and recommends HMAC SHA-256 validation before processing them. It also says to re-query the verification API and implement idempotency. [Chapa webhook documentation](https://developer.chapa.co/integrations/webhooks)

6. **No standard payment completion hook**

Mature gateways call:

```
reference_doc.run_method("on_payment_authorized", "Completed")
```

This allows the reference DocType to decide what “paid” means.

The Chapa implementation directly modifies an LMS-specific field instead.

7. **Unsafe fallback values**

The current code contains:

```
email = "mehariwamlake@gmail.com"
description = "i love you to pay"
```

These should never be production defaults.

8. **Sensitive debugging**

```
print("BODY:", response.text)
```

Payment responses can contain private information. Use `frappe.log_error()` carefully and avoid logging secrets or unnecessary customer data.

9. **Internal errors are exposed**

```
frappe.throw(frappe.get_traceback())
```

This exposes internal implementation details to the customer.

10. **Manual commit**

```
frappe.db.commit()
```

Normally Frappe commits after a successful web request. Manual commits inside business logic can leave partially updated state.

## 8. Duplicate and unused Chapa files

The repository currently has three different implementations.

### `chapa_integration.py`

[chapa\_integration.py]\(/home/anu/frappe/frappe-bench/apps/payments/payments/payment\_gateways/chapa\_integration.py) defines `ChapaGateway`, but nothing imports or uses it.

It should either become the single low-level API client or be removed.

### `chapa.py`

[chapa.py]\(/home/anu/frappe/frappe-bench/apps/payments/payments/payment\_gateways/doctype/chapa\_settings/chapa.py) is a wrapper around Chapa Settings:

```
class Chapa:
    def get_payment_url(self, **kwargs):
        return self.settings.get_payment_url(**kwargs)
```

Nothing currently uses it. The Payment Gateway already returns Chapa Settings directly.

### `chapa_checkout.py`

[chapa\_checkout.py]\(/home/anu/frappe/frappe-bench/apps/payments/payments/templates/pages/chapa\_checkout.py) contains another complete initialization implementation.

However, Chapa Settings directly returns an external Chapa checkout URL, so this page is not part of the current flow. Its accompanying [chapa\_checkout.html]\(/home/anu/frappe/frappe-bench/apps/payments/payments/templates/pages/chapa\_checkout.html) is empty.

You should have one initialization implementation, not three.

### JavaScript

[chapa\_settings.js]\(/home/anu/frappe/frappe-bench/apps/payments/payments/payment\_gateways/doctype/chapa\_settings/chapa\_settings.js) contains:

```
frappe.ui.form.on("Chapa Settings", {});
```

This registers the form but adds no client-side behavior. It is harmless but unnecessary until you need buttons or field behavior.

### Tests

[test\_chapa\_settings.py]\(/home/anu/frappe/frappe-bench/apps/payments/payments/payment\_gateways/doctype/chapa\_settings/test\_chapa\_settings.py) contains only `pass`, so the integration has no automated tests.

## 9. Recommended production architecture

Use these responsibilities:

```
Chapa Settings
├── store encrypted keys
├── validate configuration
├── initialize payment
├── verify payment through Chapa API
└── validate supported currency

Integration Request
├── unique tx_ref
├── expected amount/currency
├── reference DocType/document
├── Chapa responses
└── Queued/Completed/Failed status

Callback endpoint
├── read trx_ref
├── find Integration Request
├── call Chapa verification endpoint
├── compare all critical fields
├── complete request once
└── call reference document's on_payment_authorized()

Webhook endpoint
├── verify HMAC signature
├── acknowledge duplicate events safely
└── run the same verification/completion service
```

“Complete once” is called **idempotency**. Chapa can retry a webhook, but your system must not create two tickets, invoices or credits.

## 10. What needs to be decided before implementation

The missing piece is the business document being paid for.

For your current project, that might be an:

- Event Registration
- Ticket Order
- Booking
- Sales Invoice
- Payment Request

That DocType should implement something like:

```
def on_payment_authorized(self, status):
    if status == "Completed" and not self.payment_received:
        self.db_set("payment_received", 1)
```

Do not continue using `LMS Payment` unless your project actually installs an LMS app and that is the CTO’s intended payment document.

The next implementation step should therefore be: identify the exact DocType and fields that represent the order/payment in your project, then refactor the existing Chapa code around `Integration Request` and that document’s `on_payment_authorized()` method.