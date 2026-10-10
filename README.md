# Frappe & ERPNext Cheatsheet ⚡

The commands, APIs and snippets you reach for every day when building on [Frappe Framework](https://github.com/frappe/frappe) and [ERPNext](https://github.com/frappe/erpnext), all on one page.

> Tested against **Frappe v15**. Found a mistake or have a snippet to add? PRs are welcome. See [CONTRIBUTING.md](CONTRIBUTING.md).

⭐ **Star this repo** if it saves you a trip to the docs.

## Contents
- [Bench CLI](#bench-cli)
- [Python API (server)](#python-api-server)
- [Database queries](#database-queries)
- [Document lifecycle hooks](#document-lifecycle-hooks)
- [hooks.py](#hookspy)
- [Client scripts (JavaScript)](#client-scripts-javascript)
- [REST API](#rest-api)
- [Background jobs & scheduler](#background-jobs--scheduler)
- [Permissions](#permissions)
- [Debugging](#debugging)
- [Common errors & fixes](#common-errors--fixes)

---

## Bench CLI

### Setup
```bash
bench init frappe-bench --frappe-branch version-15   # new bench
cd frappe-bench
bench new-site mysite.local                           # create site
bench use mysite.local                                # set default site
bench get-app erpnext --branch version-15             # download app
bench --site mysite.local install-app erpnext         # install on site
bench start                                           # run dev server (:8000)
```

### Daily development
| Task | Command |
|---|---|
| Create custom app | `bench new-app my_app` |
| Apply schema changes & patches | `bench --site mysite.local migrate` |
| Rebuild JS/CSS | `bench build --app my_app` |
| Clear cache | `bench --site mysite.local clear-cache` |
| Python console | `bench --site mysite.local console` |
| MariaDB console | `bench --site mysite.local mariadb` |
| Run a function | `bench --site mysite.local execute my_app.api.fn` |
| Enable developer mode | `bench set-config -g developer_mode 1` |
| Run tests | `bench --site mysite.local run-tests --app my_app` |
| Uninstall app | `bench --site mysite.local uninstall-app my_app` |

### Updates & backups
```bash
bench update                       # pull, migrate, build all apps
bench update --reset               # discard local changes in apps
bench --site mysite.local backup --with-files
bench --site mysite.local restore path/to/db.sql.gz \
  --with-public-files files.tar --with-private-files private-files.tar
```

### Production
```bash
sudo bench setup production $USER   # nginx + supervisor
bench setup lets-encrypt mysite.com
sudo supervisorctl restart all
bench --site mysite.local set-maintenance-mode on
```

---

## Python API (server)

```python
import frappe

# Fetch a document
doc = frappe.get_doc("Sales Invoice", "SINV-0001")
doc.customer, doc.items[0].item_code

# Create
doc = frappe.get_doc({
    "doctype": "ToDo",
    "description": "Call customer",
    "allocated_to": "user@example.com",
})
doc.insert()                       # add ignore_permissions=True if needed

# Update
doc.status = "Closed"
doc.save()

# Submit / cancel (submittable doctypes)
doc.submit()
doc.cancel()

# Delete
frappe.delete_doc("ToDo", doc.name)

# New doc with defaults
doc = frappe.new_doc("Task")

# Child table row
si.append("items", {"item_code": "ITEM-001", "qty": 2})

# Single doctype value
frappe.db.get_single_value("System Settings", "country")

# Current user, session & messages
frappe.session.user
frappe.msgprint("Saved!")
frappe.throw("Qty must be positive")     # raises ValidationError
```

### Whitelisted methods (callable from client / REST)
```python
@frappe.whitelist()
def get_open_tasks(project):
    return frappe.get_list("Task", filters={"project": project, "status": "Open"},
                           fields=["name", "subject"])

@frappe.whitelist(allow_guest=True)   # no login required — validate input carefully
def ping():
    return "pong"
```

---

## Database queries

```python
# One value / several fields
frappe.db.get_value("Customer", "CUST-001", "customer_group")
frappe.db.get_value("Customer", "CUST-001", ["customer_name", "territory"], as_dict=True)

# By filters
frappe.db.get_value("Item", {"item_group": "Products", "disabled": 0}, "name")

# Lists
frappe.get_list("Task", filters={"status": "Open"}, fields=["name", "subject"],
                order_by="creation desc", limit=20)   # respects permissions
frappe.get_all("Task", filters={"status": "Open"}, pluck="name")  # ignores permissions

# Filter operators
filters = {
    "qty": [">", 10],
    "status": ["in", ["Open", "Working"]],
    "subject": ["like", "%urgent%"],
    "posting_date": ["between", ["2026-01-01", "2026-03-31"]],
    "parent_task": ["is", "not set"],
}

# Exists / count
frappe.db.exists("Customer", "CUST-001")
frappe.db.count("Task", {"status": "Open"})

# Fast field update (no hooks, no validation)
frappe.db.set_value("Task", "TASK-001", "status", "Closed")

# Query builder (preferred over raw SQL)
from frappe.query_builder import DocType
from frappe.query_builder.functions import Sum
SI = DocType("Sales Invoice")
frappe.qb.from_(SI).select(SI.customer, Sum(SI.grand_total)) \
    .where(SI.docstatus == 1).groupby(SI.customer).run(as_dict=True)

# Raw SQL — always use parameters
frappe.db.sql("select name from `tabTask` where status = %s", ("Open",), as_dict=True)

frappe.db.commit()   # needed in scripts/console; requests commit automatically
```

---

## Document lifecycle hooks

Controller methods in `my_app/my_module/doctype/my_doctype/my_doctype.py`:

| Method | Runs when |
|---|---|
| `autoname` | naming a new doc |
| `before_validate` / `validate` | before every save (put checks here) |
| `before_insert` / `after_insert` | first save only |
| `before_save` / `on_update` | every save, before / after DB write |
| `before_submit` / `on_submit` | submit |
| `before_cancel` / `on_cancel` | cancel |
| `on_update_after_submit` | edit of "Allow on Submit" fields |
| `on_trash` / `after_delete` | delete |

```python
from frappe.model.document import Document

class LibraryMember(Document):
    def validate(self):
        if self.age < 18:
            frappe.throw("Member must be an adult")

    def on_submit(self):
        frappe.get_doc({"doctype": "ToDo", "description": f"Welcome {self.name}"}).insert()
```

---

## hooks.py

```python
# Run code on another app's doctype events
doc_events = {
    "Sales Invoice": {
        "on_submit": "my_app.events.sales_invoice.on_submit",
        "validate": "my_app.events.sales_invoice.validate",
    },
    "*": {"on_update": "my_app.events.log_change"},   # every doctype
}

# Extend a doctype's form with JS
doctype_js = {"Sales Order": "public/js/sales_order.js"}

# Replace a controller class
override_doctype_class = {"ToDo": "my_app.overrides.CustomToDo"}

# Override a whitelisted method
override_whitelisted_methods = {
    "frappe.desk.doctype.event.event.get_events": "my_app.event.get_events"
}

# Ship customizations with the app (export with: bench export-fixtures)
fixtures = [
    {"dt": "Custom Field", "filters": [["module", "=", "My App"]]},
    {"dt": "Property Setter", "filters": [["module", "=", "My App"]]},
]

# Scheduler
scheduler_events = {
    "daily": ["my_app.tasks.daily_cleanup"],
    "cron": {"*/15 * * * *": ["my_app.tasks.sync"]},
}

# Global assets
app_include_js = "/assets/my_app/js/my_app.js"
```

---

## Client scripts (JavaScript)

```javascript
frappe.ui.form.on("Sales Order", {
    setup(frm) {
        // filter a Link field
        frm.set_query("customer", () => ({ filters: { disabled: 0 } }));
    },
    refresh(frm) {
        if (!frm.is_new()) {
            frm.add_custom_button("Send Reminder", () => send(frm), "Actions");
        }
        frm.toggle_display("discount_amount", frm.doc.apply_discount);
    },
    customer(frm) {                       // runs when field changes
        frm.set_value("delivery_date", frappe.datetime.add_days(frappe.datetime.nowdate(), 7));
    },
});

// Child table events: doctype = child table doctype
frappe.ui.form.on("Sales Order Item", {
    qty(frm, cdt, cdn) {
        const row = locals[cdt][cdn];
        frappe.model.set_value(cdt, cdn, "amount", row.qty * row.rate);
    },
});

// Call a whitelisted Python method
frappe.call({
    method: "my_app.api.get_open_tasks",
    args: { project: frm.doc.project },
    freeze: true,
}).then(r => console.log(r.message));

// Read a value without loading the doc
frappe.db.get_value("Customer", frm.doc.customer, "territory")
    .then(r => console.log(r.message.territory));

// UI helpers
frappe.msgprint("Done");
frappe.show_alert({ message: "Saved", indicator: "green" });
frappe.confirm("Are you sure?", () => { /* yes */ });
frappe.prompt([{ fieldname: "reason", fieldtype: "Data", reqd: 1 }],
              v => console.log(v.reason), "Reason");
frappe.set_route("Form", "Customer", "CUST-001");
```

---

## REST API

Auth header: `Authorization: token <api_key>:<api_secret>` (generate keys under **User → API Access**).

```bash
# List with filters and fields
curl -H "Authorization: token KEY:SECRET" \
  'https://site.com/api/resource/Task?filters=[["status","=","Open"]]&fields=["name","subject"]&limit_page_length=50'

# Get one
curl -H "Authorization: token KEY:SECRET" https://site.com/api/resource/Task/TASK-0001

# Create
curl -X POST -H "Authorization: token KEY:SECRET" -H "Content-Type: application/json" \
  -d '{"subject": "New task"}' https://site.com/api/resource/Task

# Update
curl -X PUT ... -d '{"status": "Completed"}' https://site.com/api/resource/Task/TASK-0001

# Delete
curl -X DELETE ... https://site.com/api/resource/Task/TASK-0001

# Call a whitelisted method
curl -H "Authorization: token KEY:SECRET" "https://site.com/api/method/my_app.api.ping"
```

---

## Background jobs & scheduler

```python
frappe.enqueue(
    "my_app.tasks.heavy_report",
    queue="long",              # short | default | long
    timeout=1500,
    enqueue_after_commit=True, # run only if the request commits
    report_name="Sales",
)

# Run after the current transaction commits
frappe.db.after_commit.add(lambda: notify())
```

```bash
bench --site mysite.local enable-scheduler
bench --site mysite.local trigger-scheduler-event daily
bench doctor                     # check workers & queues
```

---

## Permissions

```python
frappe.has_permission("Sales Invoice", "write", doc)     # returns bool
frappe.only_for("System Manager")                        # raise unless role
doc.insert(ignore_permissions=True)                      # bypass (server only)
frappe.get_roles()                                       # current user's roles

# Row-level restriction (hooks.py)
permission_query_conditions = {"Task": "my_app.perms.task_query"}
has_permission = {"Task": "my_app.perms.task_has_permission"}
```

---

## Debugging

```python
frappe.log_error(title="Sync failed", message=frappe.get_traceback())  # → Error Log doctype
frappe.logger("my_app").info("payload %s", data)                      # → logs/my_app.log
frappe.errprint(value)                                                 # → browser console (dev mode)
```
- Browser console: `cur_frm.doc`, `frappe.boot`, `frappe.session.user`
- **System Console** (Desk → System Console) to run Python on a live site
- `bench --site mysite.local console` then `%load_ext autoreload` for iterative work

---

## Common errors & fixes

| Error | Fix |
|---|---|
| Changes to JS/CSS not showing | `bench build --app my_app` and hard-refresh |
| Doctype field changes missing | `bench --site site migrate`, then `clear-cache` |
| `ModuleNotFoundError: my_app` | `bench pip install -e apps/my_app` or reinstall app |
| `CSRFTokenError` on POST | send `X-Frappe-CSRF-Token: frappe.csrf_token` or use token auth |
| Scheduler jobs not running | `bench --site site enable-scheduler`; ensure `bench start`/supervisor workers run |
| `Redis connection refused` | start bench (`bench start`) or `sudo systemctl start redis-server` |
| Site shows "Not Found" | site name must match host — use `bench use site` or `/etc/hosts` entry |
| `DocumentLockedError` | a previous job crashed holding a lock; delete `.lock` file under `sites/site/locks` |

---

## Resources
- [Frappe docs](https://docs.frappe.io/framework) · [ERPNext docs](https://docs.frappe.io/erpnext)
- [Frappe forum](https://discuss.frappe.io)
- [awesome-frappe](https://github.com/gavindsouza/awesome-frappe)

## License
[MIT](LICENSE)
