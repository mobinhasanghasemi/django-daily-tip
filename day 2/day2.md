
# 🐂 Django Daily Tip - Day 2

## ⚡ Topic: `.only()` and `.defer()` — Fetch Only What You Need

---

## ❌ Bad Code

```python
# views.py
from .models import Product

def product_list(request):
    # Pulls ALL 30 fields of Product, even the ones you don't need
    products = Product.objects.all()
    
    return render(request, 'products.html', {
        'products': products
    })
```

```html
<!-- products.html -->
{% for product in products %}
    <div class="card">
        <h3>{{ product.title }}</h3>
        <span>{{ product.price }}</span>
    </div>
{% endfor %}
```

```python
# models.py
class Product(models.Model):
    title = models.CharField(max_length=200)
    price = models.DecimalField(max_digits=10, decimal_places=2)
    description = models.TextField()           # 2000 chars of text
    specifications = models.JSONField()        # A huge JSON blob
    image_high_res = models.BinaryField()      # 5MB image
    # + 25 more fields you don't need in the list view
```

---

## 🐛 Problems with the Old Code

| Issue | Description |
|-------|-------------|
| **Wasteful Data Transfer** | `description`, `specifications`, and `image_high_res` are read from DB but never used |
| **High RAM Usage** | With 1000 products, megabytes of unused data sit in memory |
| **Slower Queries** | Reading heavy fields like `BinaryField` takes time on its own |
| **Wasted Bandwidth** | Unnecessary data travels between database and server |

---

## ✅ Optimized Code with `.only()`

```python
# views.py - Fetch only the fields you need
from django.shortcuts import render
from .models import Product

def product_list(request):
    products = Product.objects.only('title', 'price').all()
    # Only 2 out of 30 fields are read
    
    return render(request, 'products.html', {
        'products': products
    })
```

```sql
-- Generated query
SELECT id, title, price 
FROM product;
-- Instead of SELECT * with 30 columns!
```

---

## 🔄 `.defer()` — The Reverse of `.only()`

```python
# Get all fields EXCEPT the heavy binary and long description
products = Product.objects.defer(
    'description', 
    'specifications',
    'image_high_res'
).all()
```

| Method | Usage |
|--------|-------|
| `.only('a', 'b')` | Fetch **only** a and b (+ id is always included) |
| `.defer('x', 'y')` | Fetch everything **except** x and y |

---

## ⚠️ Critical Gotcha

```python
# This is a common mistake!
product = Product.objects.only('title', 'price').first()
print(product.title)   # ✅ No extra query
print(product.price)   # ✅ No extra query
print(product.sku)     # ❌ Triggers a NEW query! (sku was deferred)

# Solution: Know exactly which fields you need
product = Product.objects.only('title', 'price', 'sku').first()
```

> 🔴 **Any field NOT in `.only()` will trigger an extra query when accessed later!**

---

## 📊 Real-World Comparison

```python
import sys

# Product with 10KB description field
class Product(models.Model):
    title = models.CharField(max_length=200)
    price = models.DecimalField(max_digits=10, decimal_places=2)
    description = models.TextField()  # ~10KB per record

# Test with 1000 products
old = Product.objects.all()                # All fields
new = Product.objects.only('title', 'price')  # Just 2 fields

print(sys.getsizeof(list(old)))  # ~10MB in memory
print(sys.getsizeof(list(new)))  # ~80KB in memory
```

> 🚀 **99% reduction in memory usage!**

---

## 🧪 Daily Exercise

```python
# Optimize this code
# Assume UserProfile has: avatar (large BinaryField),
# bio (TextField), website, location, birth_date

profiles = UserProfile.objects.all()[:50]
for profile in profiles:
    print(f"{profile.user.username}: {profile.website}")
```

<details>
<summary>📖 Answer</summary>

```python
# You only need username from user and website from profile
profiles = UserProfile.objects.select_related('user')\
    .only('website', 'user__username')[:50]

for profile in profiles:
    print(f"{profile.user.username}: {profile.website}")
    # Without reading bio, avatar, or birth_date
```
</details>

---

## 📌 Golden Tip of the Day

> **`.only()` and `.defer()` take optimization to the next level — not just reducing query count, but reducing data volume too.**

**When to use:**
- List views showing only summaries (store page, admin tables)
- Heavy file fields (attached BinaryField, ImageField)
- APIs returning lightweight JSON responses

**The Ultimate Django Performance Formula:**
```
select_related/prefetch_related  +  only/defer  =  Absolute Power
```
