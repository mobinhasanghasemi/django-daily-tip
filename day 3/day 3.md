

# 🐂 Django Daily Tip - Day 3

## ⚡ Topic: `values()` and `values_list()` — Dictionaries Instead of Model Instances

---

## ❌ Bad Code

```python
# views.py
from django.shortcuts import render
from .models import Order

def sales_report(request):
    # Pulls full model instances with all their overhead
    orders = Order.objects.select_related('customer', 'product').all()
    
    report = []
    for order in orders:
        report.append({
            'id': order.id,
            'customer': order.customer.name,
            'product': order.product.title,
            'total': order.product.price * order.quantity,
            'date': order.created_at.date(),
        })
    
    return render(request, 'report.html', {'report': report})
```

---

## 🐛 Problems with the Old Code

| Issue | Description |
|-------|-------------|
| **Model Instance Overhead** | Each `order` object carries full ORM machinery: signals, lazy loading, relation caches, model methods |
| **Memory Waste** | 10,000 orders × ~2KB per instance = ~20MB just for Python objects |
| **Slower Iteration** | Creating Django model instances is expensive — `__init__`, field setup, descriptor binding |
| **Unused Features** | You don't need `.save()`, `.delete()`, signals, or model methods in a report |

---

## ✅ Optimized Code with `values()`

```python
# views.py - Lightweight dictionaries instead of model instances
from django.shortcuts import render
from .models import Order

def sales_report(request):
    # Returns dictionaries, NOT model instances
    report = Order.objects.select_related('customer', 'product')\
        .values(
            'id',
            'customer__name',     # Double-underscore to follow relations!
            'product__title',
            'product__price',
            'quantity',
            'created_at'
        )
    
    # Data is pre-flattened, just iterate
    formatted_report = []
    for item in report:
        formatted_report.append({
            'id': item['id'],
            'customer': item['customer__name'],
            'product': item['product__title'],
            'total': item['product__price'] * item['quantity'],
            'date': item['created_at'].date(),
        })
    
    return render(request, 'report.html', {'report': formatted_report})
```

**What changed?**
- No model instances created — just plain Python dicts
- Relations followed with `__` syntax: `customer__name`, `product__title`
- You still need `select_related()` because `values()` follows the JOIN it generates

---

## 🔄 `values_list()` — Even Lighter, Flat Tuples

```python
# Returns tuples — perfect for simple lists
# --------------------------------------------------

# Single field as a flat list (flat=True)
product_names = Product.objects.filter(active=True)\
    .values_list('title', flat=True)
# Result: ['Product A', 'Product B', 'Product C']
# Perfect for: dropdown choices, autocomplete, simple lists


# Multiple fields as tuples
price_list = Product.objects.values_list('title', 'price')
# Result: [('Product A', 50000), ('Product B', 75000), ...]
# Perfect for: CSV export, feeding into charts


# Named tuples for readability
price_list = Product.objects.values_list('title', 'price', named=True)
for item in price_list:
    print(item.title, item.price)  # Instead of item[0], item[1]
```

---

## 🎯 When to Use Which

```python
# ❌ Model Instances (default)
# Use when you need:
# - Model methods (product.calculate_discount())
# - Signals to fire
# - Save/update/delete records
# - Custom properties on the model
products = Product.objects.all()


# ✅ values() 
# Use when you need:
# - Display-only data (reports, APIs, admin tables)
# - Dictionary-like access by key name
# - To flatten related fields (user__email)
products = Product.objects.values('title', 'price', 'category__name')


# ✅ values_list()
# Use when you need:
# - Simple flat list of one field (flat=True)
# - Tuples ready for CSV/Excel export
# - Feeding data into charts or pandas
# - A list of IDs
products = Product.objects.values_list('id', 'title', 'price')
```

---

## 🔥 Real-World Scenario: Dropdown & Export

```python
# models.py
class City(models.Model):
    name = models.CharField(max_length=100)
    province = models.CharField(max_length=100)
    population = models.IntegerField()

# --------------------------------------------------
# Use Case 1: Dropdown in a form
# --------------------------------------------------
def get_city_choices():
    # One query, flat list of (id, name) tuples
    return City.objects.values_list('id', 'name').order_by('name')
    # Result: [(1, 'Karaj'), (2, 'Mashhad'), (3, 'Tehran'), ...]


# --------------------------------------------------
# Use Case 2: Export to CSV
# --------------------------------------------------
import csv

def export_cities_csv():
    cities = City.objects.values_list('name', 'province', 'population')
    
    with open('cities.csv', 'w') as f:
        writer = csv.writer(f)
        writer.writerow(['Name', 'Province', 'Population'])  # Header
        writer.writerows(cities)  # All data in one shot — blazing fast
    # No loop, no model instantiation, just raw tuples into CSV


# --------------------------------------------------
# Use Case 3: Simple list of names
# --------------------------------------------------
def city_autocomplete():
    # One query, returns a flat list of strings
    return list(City.objects.filter(
        population__gt=100000
    ).values_list('name', flat=True))
    # Result: ['Tehran', 'Mashhad', 'Isfahan', 'Karaj', ...]
```

---

## 📊 Performance Comparison

| Metric | Model Instances | values() |
|--------|-----------------|----------|
| 1,000 records | ~2MB RAM — 0.3s | ~10KB RAM — 0.02s |
| 10,000 records | ~20MB RAM — 2.8s | ~50KB RAM — 0.08s |
| 100,000 records | ~200MB RAM — 💀 | ~200KB RAM — 0.7s |

> 🚀 **Up to 1000x less RAM and 30x faster!**

---

## ⚠️ Important Gotchas

```python
# Gotcha 1: values() still needs select_related for JOINs
# ❌ Wrong — this will do N+1 queries if you access relations
orders = Order.objects.values('id', 'customer__name')
# Each customer__name access might hit the DB separately!

# ✅ Correct — tell it to JOIN
orders = Order.objects.select_related('customer')\
    .values('id', 'customer__name')


# Gotcha 2: values() returns dicts, not objects
order = Order.objects.values('id', 'quantity').first()
print(type(order))  # <class 'dict'>
print(order.id)     # ❌ AttributeError!
print(order['id'])  # ✅ Dict access


# Gotcha 3: No model methods available
product = Product.objects.values('price').first()
# product.calculate_discount()  # ❌ Doesn't exist on dict!
```

---

## 🧪 Daily Exercise

```python
# Optimize this code using values/values_list
# We just need category names and product counts for a Django template chart
# No model methods needed

categories = Category.objects.prefetch_related('products').all()
data = []
for cat in categories:
    data.append({
        'name': cat.name,
        'count': cat.products.count(),  # This triggers an extra query per category!
    })
# With 50 categories = 1 + 50 queries
```

<details>
<summary>📖 Answer (Think first!)</summary>

```python
# We'll keep it simple — just values() and a manual count in Python
# (We haven't learned annotate yet — that's for another day!)

categories = Category.objects.prefetch_related('products')\
    .values('id', 'name')

data = []
for cat in categories:
    # Still need to count products, but at least categories are light dicts
    product_count = Product.objects.filter(category_id=cat['id']).count()
    data.append({
        'name': cat['name'],
        'count': product_count,
    })

# Better: Use values_list to get just the IDs we need
category_ids = Category.objects.values_list('id', flat=True)
# Then do one query per category (still not perfect, but lighter objects)
```
</details>

---

## 📌 Golden Tip of the Day

> **Django model instances are heavy. When you only need to *read* and *display* data, use `values()` or `values_list()`. Save model instances for when you actually need to `.save()` or use model methods.**

| Method | Returns | Memory per 10K rows | Use Case |
|--------|---------|---------------------|----------|
| `.all()` | Model instances | ~20MB | CRUD operations |
| `.values()` | Dictionaries | ~50KB | Reports, APIs, display |
| `.values_list()` | Tuples | ~30KB | Exports, charts, lists |

---
