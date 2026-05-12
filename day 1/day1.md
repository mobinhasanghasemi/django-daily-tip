
# 🐂 Django Daily Tip - Day 1

## ⚡ Topic: `select_related` vs `prefetch_related`

---

## ❌ Bad/Old Code (Without select_related)

```python
# views.py
from django.shortcuts import render
from .models import Book

def book_list(request):
    books = Book.objects.all()
    
    context = []
    for book in books:
        context.append({
            'title': book.title,
            'author_name': book.author.name,  # Problem right here
            'publisher_name': book.author.publisher.name,  # Deeper problem
        })
    
    return render(request, 'books.html', {'books': context})
```

```python
# models.py
class Publisher(models.Model):
    name = models.CharField(max_length=100)

class Author(models.Model):
    name = models.CharField(max_length=100)
    publisher = models.ForeignKey(Publisher, on_delete=models.CASCADE)

class Book(models.Model):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(Author, on_delete=models.CASCADE)
```

---

## 🐛 Bugs and Drawbacks of the Old Code

| Issue | Description |
|-------|-------------|
| **N+1 Query Problem** | For 10 books: 1 query to fetch books + 10 queries for each book's author = **11 queries** |
| **Deeper N+1 Problem** | On `book.author.publisher.name` line, an additional query hits the Publisher table for each book → **1 + (10×2) = 21 queries** |
| **Severe Slowdown** | With 1000 books, approximately 2001 queries hit the database |
| **No Scalability** | Performance degrades drastically as the database grows |
| **Increased Database Load** | Consecutive connections and queries overwhelm the database |

---

## ✅ Optimized Code (With select_related)

```python
# views.py - Optimized version
from django.shortcuts import render
from django.db.models import Prefetch
from .models import Book

def book_list(request):
    # Loads all foreign key relationships with a SINGLE query
    books = Book.objects.select_related(
        'author__publisher'  # This means: also fetch author AND their publisher
    ).all()
    
    context = []
    for book in books:
        context.append({
            'title': book.title,
            'author_name': book.author.name,           # No extra query
            'publisher_name': book.author.publisher.name,  # Still no extra query
        })
    
    return render(request, 'books.html', {'books': context})
```

### Generated SQL (Just 1 Query):

```sql
SELECT 
    book.id,
    book.title,
    book.author_id,
    author.id,
    author.name,
    author.publisher_id,
    publisher.id,
    publisher.name
FROM book
INNER JOIN author ON book.author_id = author.id
INNER JOIN publisher ON author.publisher_id = publisher.id
```

---

## 📊 Performance Comparison

| Number of Records | Old Code (Query Count) | New Code (Query Count) |
|-------------------|------------------------|------------------------|
| 10 books | 21 queries | **1 query** |
| 100 books | 201 queries | **1 query** |
| 1000 books | 2001 queries | **1 query** |

> 🚀 **2000x reduction in queries for 1000 books!**

---

## 🔥 Real-World Example You'll Use a Lot

### Scenario: Store Order Admin Panel

```python
# models.py
class Customer(models.Model):
    name = models.CharField(max_length=100)
    phone = models.CharField(max_length=15)

class Product(models.Model):
    title = models.CharField(max_length=200)
    price = models.DecimalField(max_digits=10, decimal_places=2)

class Order(models.Model):
    customer = models.ForeignKey(Customer, on_delete=models.CASCADE)
    product = models.ForeignKey(Product, on_delete=models.CASCADE)
    quantity = models.IntegerField()
    created_at = models.DateTimeField(auto_now_add=True)

# views.py - Old format (BAD)
def old_order_list(request):
    orders = Order.objects.all()  # Just Order
    result = []
    for order in orders:
        result.append({
            'customer': order.customer.name,   # +1 query
            'product': order.product.title,     # +1 query
            'total': order.product.price * order.quantity,  # product again
        })
    # For 500 orders = 1 + (500*2) = 1001 queries
    return render(request, 'orders.html', {'orders': result})

# views.py - New format (NICE)
def new_order_list(request):
    orders = Order.objects.select_related('customer', 'product').all()
    # Just 1 query for the entire operation 🎯
    
    result = []
    for order in orders:
        result.append({
            'customer': order.customer.name,
            'product': order.product.title,
            'total': order.product.price * order.quantity,
        })
    return render(request, 'orders.html', {'orders': result})
```

---

## 📌 Golden Tip of the Day

> **`select_related`** is used for `ForeignKey` and `OneToOneField` relationships and fetches everything with a SQL `JOIN` in **a single query**.

> **`prefetch_related`** is used for `ManyToManyField` and `reverse ForeignKey` relationships and works with **two queries** (then joins in Python).

**Rule of Thumb:**
- Expecting to need related objects? → Always use `select_related` or `prefetch_related`.
- Don't use `len(queryset)` to check count (it executes the query). Use `.count()` instead.

---

## 🧪 Daily Exercise

Optimize the following code (assuming `Post`, `User`, and `Profile` models):

```python
# Before
posts = Post.objects.filter(is_published=True)
for post in posts:
    print(post.user.profile.bio)
    print(post.user.email)
```

<details>
<summary>📖 Answer (Think before looking)</summary>

```python
posts = Post.objects.filter(is_published=True).select_related('user__profile')
for post in posts:
    print(post.user.profile.bio)   # No more extra queries
    print(post.user.email)
```
</details>

---

**Remember:** Every dot-chained access (`book.author.publisher`) can be a hidden query. `select_related` is your magic sword for this.
