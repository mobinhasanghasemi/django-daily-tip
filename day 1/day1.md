بسیار عالی. طبق درخواست شما، **روز اول از Django Daily Tips** رو به سبک «کد نادرست و قدیمی → بررسی باگ‌ها و معایب → کد بهینه → مثال واقعی» آماده می‌کنم.

---

# 🐂 Django Daily Tip - Day 1

## ⚡ موضوع: `select_related` در مقابل `prefetch_related`

---

## ❌ کد نادرست و قدیمی (بدون استفاده از select_related)

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
            'author_name': book.author.name,  # مشکل اینجاست
            'publisher_name': book.author.publisher.name,  # مشکل عمیق‌تر
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

## 🐛 باگ‌ها و معایب کد قدیمی

| مشکل | توضیح |
|------|------|
| **مشکل N+1 Query** | برای ۱۰ کتاب، ابتدا ۱ کوئری برای گرفتن کتاب‌ها + ۱۰ کوئری برای گرفتن نویسنده هر کتاب = **۱۱ کوئری** |
| **مشکل N+1 عمیق‌تر** | در خط `book.author.publisher.name`، برای هر کتاب یک کوئری دیگر هم به جدول Publisher می‌زنه → **۱ + (۱۰×۲) = ۲۱ کوئری** |
| **کند شدن شدید** | با ۱۰۰۰ کتاب، حدود ۲۰۰۱ کوئری به دیتابیس زده می‌شه |
| **عدم مقیاس‌پذیری** | با رشد دیتابیس، سرعت به شدت افت می‌کند |
| **افزایش بار دیتابیس** | تعداد اتصالات و کوئری‌های پیدرپی، دیتابیس رو اذیت می‌کند |

---

## ✅ کد بهینه (با select_related)

```python
# views.py - نسخه بهینه
from django.shortcuts import render
from django.db.models import Prefetch
from .models import Book

def book_list(request):
    # با یک کوئری، همه روابط خارجی رو یکجا load می‌کنه
    books = Book.objects.select_related(
        'author__publisher'  # این یعنی author و publisher اون author رو هم بگیر
    ).all()
    
    context = []
    for book in books:
        context.append({
            'title': book.title,
            'author_name': book.author.name,           # بدون کوئری اضافه
            'publisher_name': book.author.publisher.name,  # باز هم بدون کوئری اضافه
        })
    
    return render(request, 'books.html', {'books': context})
```

### خروجی کوئری تولید شده (فقط ۱ کوئری):

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

## 📊 مقایسه عملکرد

| تعداد رکوردها | کد قدیمی (تعداد کوئری) | کد جدید (تعداد کوئری) |
|--------------|------------------------|----------------------|
| ۱۰ کتاب | ۲۱ کوئری | **۱ کوئری** |
| ۱۰۰ کتاب | ۲۰۱ کوئری | **۱ کوئری** |
| ۱۰۰۰ کتاب | ۲۰۰۱ کوئری | **۱ کوئری** |

> 🚀 **کاهش ۲۰۰۰ برابر تعداد کوئری‌ها برای ۱۰۰۰ کتاب!**

---

## 🔥 مثال موقعیتی که خیلی استفاده می‌شه

### سناریو: صفحه پنل مدیریت سفارشات فروشگاه

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

# views.py - فرمت قدیمی (بد)
def old_order_list(request):
    orders = Order.objects.all()  # فقط Order
    result = []
    for order in orders:
        result.append({
            'customer': order.customer.name,   # +1 query
            'product': order.product.title,     # +1 query
            'total': order.product.price * order.quantity,  # بازهم product
        })
    # برای 500 سفارش = 1 + (500*2) = 1001 کوئری
    return render(request, 'orders.html', {'orders': result})

# views.py - فرمت جدید (نایس)
def new_order_list(request):
    orders = Order.objects.select_related('customer', 'product').all()
    # فقط 1 کوئری کل عملیات 🎯
    
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

## 📌 نکته طلایی روز

> **`select_related`** برای رابطه `ForeignKey` و `OneToOneField` استفاده می‌شه و با `JOIN` داخل **یک کوئری** همه رو میاره.

> **`prefetch_related`** برای رابطه `ManyToManyField` و `reverse ForeignKey` استفاده می‌شه و با **دو کوئری** (و بعد join در پایتون) کار می‌کنه.

**قاعده سرانگشتی:**
- پیش‌بینی می‌کنی به رابطه‌های خارجی نیاز داری؟ → حتماً `select_related` یا `prefetch_related` بزن.
- از `len(queryset)` برای چک کردن تعداد استفاده نکن (اون کوئری رو execute می‌کنه). از `.count()` استفاده کن.

---

## 🧪 تمرین روز

کد زیر رو بهینه کن (با فرض مدل‌های `Post` و `User` و `Profile`):

```python
# قبل
posts = Post.objects.filter(is_published=True)
for post in posts:
    print(post.user.profile.bio)
    print(post.user.email)
```

<details>
<summary>📖 جواب (قبل از نگاه کردن فکر کن)</summary>

```python
posts = Post.objects.filter(is_published=True).select_related('user__profile')
for post in posts:
    print(post.user.profile.bio)   # دیگه کوئری اضافه نداره
    print(post.user.email)
```
</details>

---

**یادت باشه:** هر حرف‌نقطه‌دار (`book.author.publisher`) می‌تونه یه کوئری مخفی باشه. `select_related` شمشیر جادویی‌اته براش.

---

بعد از خوندن این تیپ، کدت از نظر تعداد کوئری‌ها چند برابر سریع‌تر شد؟ 😎