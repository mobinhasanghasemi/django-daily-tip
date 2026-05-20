

\# 🐂 Django Daily Tip - Day 3



\## ⚡ Topic: `values()` and `values\_list()` — Dictionaries Instead of Model Instances



---



\## ❌ Bad Code



```python

\# views.py

from django.shortcuts import render

from .models import Order



def sales\_report(request):

&nbsp;   # Pulls full model instances with all their overhead

&nbsp;   orders = Order.objects.select\_related('customer', 'product').all()

&nbsp;   

&nbsp;   report = \[]

&nbsp;   for order in orders:

&nbsp;       report.append({

&nbsp;           'id': order.id,

&nbsp;           'customer': order.customer.name,

&nbsp;           'product': order.product.title,

&nbsp;           'total': order.product.price \* order.quantity,

&nbsp;           'date': order.created\_at.date(),

&nbsp;       })

&nbsp;   

&nbsp;   return render(request, 'report.html', {'report': report})

```



---



\## 🐛 Problems with the Old Code



| Issue | Description |

|-------|-------------|

| \*\*Model Instance Overhead\*\* | Each `order` object carries full ORM machinery: signals, lazy loading, relation caches, model methods |

| \*\*Memory Waste\*\* | 10,000 orders × ~2KB per instance = ~20MB just for Python objects |

| \*\*Slower Iteration\*\* | Creating Django model instances is expensive — `\_\_init\_\_`, field setup, descriptor binding |

| \*\*Unused Features\*\* | You don't need `.save()`, `.delete()`, signals, or model methods in a report |



---



\## ✅ Optimized Code with `values()`



```python

\# views.py - Lightweight dictionaries instead of model instances

from django.shortcuts import render

from .models import Order



def sales\_report(request):

&nbsp;   # Returns dictionaries, NOT model instances

&nbsp;   report = Order.objects.select\_related('customer', 'product')\\

&nbsp;       .values(

&nbsp;           'id',

&nbsp;           'customer\_\_name',     # Double-underscore to follow relations!

&nbsp;           'product\_\_title',

&nbsp;           'product\_\_price',

&nbsp;           'quantity',

&nbsp;           'created\_at'

&nbsp;       )

&nbsp;   

&nbsp;   # Data is pre-flattened, just iterate

&nbsp;   formatted\_report = \[]

&nbsp;   for item in report:

&nbsp;       formatted\_report.append({

&nbsp;           'id': item\['id'],

&nbsp;           'customer': item\['customer\_\_name'],

&nbsp;           'product': item\['product\_\_title'],

&nbsp;           'total': item\['product\_\_price'] \* item\['quantity'],

&nbsp;           'date': item\['created\_at'].date(),

&nbsp;       })

&nbsp;   

&nbsp;   return render(request, 'report.html', {'report': formatted\_report})

```



\*\*What changed?\*\*

\- No model instances created — just plain Python dicts

\- Relations followed with `\_\_` syntax: `customer\_\_name`, `product\_\_title`

\- You still need `select\_related()` because `values()` follows the JOIN it generates



---



\## 🔄 `values\_list()` — Even Lighter, Flat Tuples



```python

\# Returns tuples — perfect for simple lists

\# --------------------------------------------------



\# Single field as a flat list (flat=True)

product\_names = Product.objects.filter(active=True)\\

&nbsp;   .values\_list('title', flat=True)

\# Result: \['Product A', 'Product B', 'Product C']

\# Perfect for: dropdown choices, autocomplete, simple lists





\# Multiple fields as tuples

price\_list = Product.objects.values\_list('title', 'price')

\# Result: \[('Product A', 50000), ('Product B', 75000), ...]

\# Perfect for: CSV export, feeding into charts





\# Named tuples for readability

price\_list = Product.objects.values\_list('title', 'price', named=True)

for item in price\_list:

&nbsp;   print(item.title, item.price)  # Instead of item\[0], item\[1]

```



---



\## 🎯 When to Use Which



```python

\# ❌ Model Instances (default)

\# Use when you need:

\# - Model methods (product.calculate\_discount())

\# - Signals to fire

\# - Save/update/delete records

\# - Custom properties on the model

products = Product.objects.all()





\# ✅ values() 

\# Use when you need:

\# - Display-only data (reports, APIs, admin tables)

\# - Dictionary-like access by key name

\# - To flatten related fields (user\_\_email)

products = Product.objects.values('title', 'price', 'category\_\_name')





\# ✅ values\_list()

\# Use when you need:

\# - Simple flat list of one field (flat=True)

\# - Tuples ready for CSV/Excel export

\# - Feeding data into charts or pandas

\# - A list of IDs

products = Product.objects.values\_list('id', 'title', 'price')

```



---



\## 🔥 Real-World Scenario: Dropdown \& Export



```python

\# models.py

class City(models.Model):

&nbsp;   name = models.CharField(max\_length=100)

&nbsp;   province = models.CharField(max\_length=100)

&nbsp;   population = models.IntegerField()



\# --------------------------------------------------

\# Use Case 1: Dropdown in a form

\# --------------------------------------------------

def get\_city\_choices():

&nbsp;   # One query, flat list of (id, name) tuples

&nbsp;   return City.objects.values\_list('id', 'name').order\_by('name')

&nbsp;   # Result: \[(1, 'Karaj'), (2, 'Mashhad'), (3, 'Tehran'), ...]





\# --------------------------------------------------

\# Use Case 2: Export to CSV

\# --------------------------------------------------

import csv



def export\_cities\_csv():

&nbsp;   cities = City.objects.values\_list('name', 'province', 'population')

&nbsp;   

&nbsp;   with open('cities.csv', 'w') as f:

&nbsp;       writer = csv.writer(f)

&nbsp;       writer.writerow(\['Name', 'Province', 'Population'])  # Header

&nbsp;       writer.writerows(cities)  # All data in one shot — blazing fast

&nbsp;   # No loop, no model instantiation, just raw tuples into CSV





\# --------------------------------------------------

\# Use Case 3: Simple list of names

\# --------------------------------------------------

def city\_autocomplete():

&nbsp;   # One query, returns a flat list of strings

&nbsp;   return list(City.objects.filter(

&nbsp;       population\_\_gt=100000

&nbsp;   ).values\_list('name', flat=True))

&nbsp;   # Result: \['Tehran', 'Mashhad', 'Isfahan', 'Karaj', ...]

```



---



\## 📊 Performance Comparison



| Metric | Model Instances | values() |

|--------|-----------------|----------|

| 1,000 records | ~2MB RAM — 0.3s | ~10KB RAM — 0.02s |

| 10,000 records | ~20MB RAM — 2.8s | ~50KB RAM — 0.08s |

| 100,000 records | ~200MB RAM — 💀 | ~200KB RAM — 0.7s |



> 🚀 \*\*Up to 1000x less RAM and 30x faster!\*\*



---



\## ⚠️ Important Gotchas



```python

\# Gotcha 1: values() still needs select\_related for JOINs

\# ❌ Wrong — this will do N+1 queries if you access relations

orders = Order.objects.values('id', 'customer\_\_name')

\# Each customer\_\_name access might hit the DB separately!



\# ✅ Correct — tell it to JOIN

orders = Order.objects.select\_related('customer')\\

&nbsp;   .values('id', 'customer\_\_name')





\# Gotcha 2: values() returns dicts, not objects

order = Order.objects.values('id', 'quantity').first()

print(type(order))  # <class 'dict'>

print(order.id)     # ❌ AttributeError!

print(order\['id'])  # ✅ Dict access





\# Gotcha 3: No model methods available

product = Product.objects.values('price').first()

\# product.calculate\_discount()  # ❌ Doesn't exist on dict!

```



---



\## 🧪 Daily Exercise



```python

\# Optimize this code using values/values\_list

\# We just need category names and product counts for a Django template chart

\# No model methods needed



categories = Category.objects.prefetch\_related('products').all()

data = \[]

for cat in categories:

&nbsp;   data.append({

&nbsp;       'name': cat.name,

&nbsp;       'count': cat.products.count(),  # This triggers an extra query per category!

&nbsp;   })

\# With 50 categories = 1 + 50 queries

```



<details>

<summary>📖 Answer (Think first!)</summary>



```python

\# We'll keep it simple — just values() and a manual count in Python

\# (We haven't learned annotate yet — that's for another day!)



categories = Category.objects.prefetch\_related('products')\\

&nbsp;   .values('id', 'name')



data = \[]

for cat in categories:

&nbsp;   # Still need to count products, but at least categories are light dicts

&nbsp;   product\_count = Product.objects.filter(category\_id=cat\['id']).count()

&nbsp;   data.append({

&nbsp;       'name': cat\['name'],

&nbsp;       'count': product\_count,

&nbsp;   })



\# Better: Use values\_list to get just the IDs we need

category\_ids = Category.objects.values\_list('id', flat=True)

\# Then do one query per category (still not perfect, but lighter objects)

```

</details>



---



\## 📌 Golden Tip of the Day



> \*\*Django model instances are heavy. When you only need to \*read\* and \*display\* data, use `values()` or `values\_list()`. Save model instances for when you actually need to `.save()` or use model methods.\*\*



| Method | Returns | Memory per 10K rows | Use Case |

|--------|---------|---------------------|----------|

| `.all()` | Model instances | ~20MB | CRUD operations |

| `.values()` | Dictionaries | ~50KB | Reports, APIs, display |

| `.values\_list()` | Tuples | ~30KB | Exports, charts, lists |



