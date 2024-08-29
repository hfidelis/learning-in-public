# Ways to optimize queries in Django 🐍🚀

We can make our operations more performant using Django's own methods, generally request delays are consequence from multiple JOINS performed inside queries, result from model's relationships.

## 1. Using `prefetch_related()` and `select_related()`

When we define the ***queryset*** of a ***view*** we can perform ***JOINS*** in advance within it. Thus, we reduce the number of operations that will be realized later.

**Example:** Within the ***serializer*** used by ***view*** we need data from some entity related to the model, to fill in a field for example. So for each instance that will be serialized, a set of operations will be performed to bring this data. When performing these operations in ***view*** when defining the ***queryset***, all operations will be done in a single shot, carrying all needed data to serializer.

### `select_related()`: Used for 1-to-1 `OneToOneField` or `ForeignKey` relationships.

### `prefetch_related()`: Using for relationships where we will have several objects, such as `ManyToManyField` or reverse `ForeignKey` accesses.

### Example:

```python
# models.py

class Company(models.Model):
	name = models.CharField(max_length=100, unique=True, blank=False, null=False)
													
class Employee(models.Model):
	name = models.CharField(max_length=100, unique=True, blank=False, null=False)
													
	on_vacation = models.BooleanField(default=False)
	
	company = models.ForeignKey(Company, on_delete=models.PROTECT, blank=True, null=True)
```

### In the `Employee` entity serializer we look for the *name* of `Company` through the *FK*, each time this *serializer* receives an instance to *serialize*, it will do a *JOIN* with ` Company` to fetch the data.

```python
# serializers.py

class EmployeeSerializer(serializers.ModelSerializer):
	company_name = serializers.CharField(source='company.name', read_only=True)
	
	class Meta:
		model = Employee
		fields = [
			'id',
			'name',
			'company_name',
		]
```

### In the `Employee` entity *view* we use `select_related()` passing the *FK* field to `Company`, thus an *INNER JOIN* is performed in the *queryset*, without the need to be performed individually in the serializer.

```python
# views.py

class EmployeeViewSet(ModelViewSet):
	queryset = Employee.objects.select_related('company').order_by('pk')
```

### With `prefetch_related()` we can do this when we have bigger relationships.

**In this case, `employee_set` (or the label defined in FK `related_name`) was passed as a parameter, listing the `Employees` of each `Company`.**

```python
queryset = Company.objects.prefetch_related('employee_set').order_by('pk')
```

### Practical Example:

```python
queryset = Order.objects.select_related(
                                    'responsible',
                                    'client',
                                    'company',
                                    'unit',
                                ) \
                                .prefetch_related(
                                    'contract__products',
                                    'contract__participants',
                                ) \
                                .order_by('pk')
```

**In the `Order` entity we are performing the *JOIN* with its foreign keys `['contract', 'interest', 'company', 'owner']`, and a `prefetch_related()` with several instances accessed through the `'__'` lookups of Django fields.**

## 2. Avoiding loops and using methods like `aggregate()` and `update()`

We can avoid building loops using some alternatives, depending on the context.

### `aggregate()`

**Inserts a field into the *queryset* and finally returns a dictionary `(dict)`.**

### Example:

- Using `loop`
    
    ```python
    price = 0
    items = Item.objects.all()
    
    for item in items:
        total_price += item.price
    ```
    
- Using `aggregate()`
    
    ```python
    from django.db.models import Sum
    
    items = Item.objects.all().aggregate(total_price=Sum('price'))
    
    total_price = items.get('total_price', 0)
    ```

### Both ways are iterating over the objects from Item entity and adding their price field, however using `aggregate` we are performing a more performant operation due to several Django reasons and also database optimizations.

### Practical Example:

```python
# serializers.py

contract_price = serializers.SerializerMethodField()

def get_contract_price(self, obj):
	ALLOWED_TYPES = ['product', 'service']

	price = Product.objects.filter(
                                contract=obj.id,
                                item__type__in=ALLOWED_TYPES
                            ).aggregate(
                                total_price=Sum(F('price') * F('quantity'))
                            ).get('total_price', 0)

	return price
```

**Here we are filtering a set of objects, then we are aggregating in the new field `total_price` the values ​from `price` field that is present in the primary key object of the `item` field. We perform aggregation with the `Sum()` operator and the `F()` operator, `F()` converts a field to be used in operations.**

### `update()`

### In the same way that `aggregate` is more performant than loops, making some changes in objects using `update()` becomes more performant than changing directly from object's fields.

### Example:

- Accessing fields

```python
items = Items.objects.all()

my_item = items.get(pk=15)
my_item.name = 'Teste'
my_item.save()
```

- Using `update()`

```python
items = Items.objects.all()

# 1. Updating a single item

items.filter(pk=15).update(name='Teste')

# 2. Updating multiple items

# By passing a list of primary keys
items.filter(pk__in=[15, 16, 17]).update(name='Teste')

# By passing a filter
items.filter(category='technology').update(category='tech')
```

### Both ways are updating the item's `name` field with `pk=15`, but the second becomes more performant for bulk updates or when we don't need to directly access the field. The first mode is more necessary when we need to perform some calc or logic.

## 3. Creating a `serializer` for each specific context

**If a `view` will not use all the fields from a model or you need to *serialize* an object within another `serializer`, and these fields require some effort to be *serialized*, it is worth creating a `serializer ` model-specific ` view `, using only what you will need.**

### Example:

```python
class ProductSerializer(serializers.ModelSerializer):
	class Meta:
		model = Product
		fields = '__all__'

class ProductDatesSerializer(serializers.ModelSerializer):
	class Meta:
		model = Product
		fields = [
			'id',
			'expiration_date',
			'manufacturing_date'.
		]

class SimpleProductSerializer(serializers.ModelSerializer):
	class Meta:
		model = Product
		fields = [
			'id',
			'name',
			'price',
		]

class ComplexProductSerializer(serializers.ModelSerializer):
	supplier_set = SupplierSerializer(source='supplier', read_only=True)
																		
	brand_set = BrandSerializer(source='supplier', read_only=True)
															
	ingredient_set = IngredientSerializer(source='ingredients', many=True, read_only=True)

	class Meta:
		model = Product
		fields = [
			'id',
			'name',
			'price',
			'brand'
			'brand_set',
			'supplier',
			'supplier_set',
			'ingredient_set',
			'expiration_date',
			'manufacturing_date',
		]
```

### `Product` contains several serializers with clearly defined objectives, it is noticeable that the last `(ComplexProductSerializer)` generates a high volume of data, making it less performant, so in a situation where I only need the basics, I can use an existing one or create a new one with the necessary fields.