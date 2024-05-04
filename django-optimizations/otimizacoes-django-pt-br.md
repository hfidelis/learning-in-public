# Maneiras de otimizar queries com Django

Podemos tornar nossas operações mais performáticas utilizando métodos do próprio Django, geralmente os atraso são consequências de múltiplos JOINS realizados nas queries, frutos de relações entre os modelos.

## Utilizando `prefetch_related()` e `select_related()`

Quando definimos o ***queryset*** de uma ***view*** podemos realizar os ***JOINS*** de forma prévia dentro da mesma. Assim, reduzimos o número de operações que vão ser realizadas posteriormente. 

**Exemplo:** Dentro do ***serializer*** utilizado pela ***view*** precisamos de dados de entidades relacionadas com o modelo, para preencher um campo por exemplo. Assim para cada instância que será serializada, um conjunto de operações será realizada para trazer este dado. Ao realizar estas operações na ***view*** no momento de definir o ***queryset***, todas as operações serão feitas em uma única tacada, levando os dados para serem apenas serializados.

### `select_related()`: Utilizado para relações 1 para 1 OneToOneField ou chaves estrangeiras ForeignKey.

### `prefetch_related()`: Utilizando para relações onde vamos ter vários objetos, como ManyToManyField ou em acessos reversos de chaves estrangeiras.

### Exemplo:

```python
# models.py

class Company(models.Model):
	name = models.CharField(max_length=100, unique=True, blank=False, null=False)
													
class Employee(models.Model):
	name = models.CharField(max_length=100, unique=True, blank=False, null=False)
													
	on_vacation = models.BooleanField(default=False)
	
	company = models.ForeignKey(Company, on_delete=models.PROTECT, blank=True, null=True)
```

### No serializer da entidade `Employee` nós buscamos o *name* de `Company` através da *FK*, cada vez que este *serializer* receber uma instância para *serializar*, ele vai fazer um *JOIN* com `Company` para buscar o dado.

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

### Na *view* da entidade `Employee` nós usamos o `select_related()` passando o campo da *FK* para `Company`, assim é realizado um *INNER JOIN* no *queryset*, sem necessidade de ser realizado individualmente no serializer.

```python
# views.py

class EmployeeViewSet(ModelViewSet):
	queryset = Employee.objects.select_related('company').order_by('pk')
```

### Já com o `prefetch_related()` podemos fazer isso quando a relação for maior.

**Neste caso foi passado como parâmetro `employee_set`  (ou o nome que estivesse definido no `related_name` da FK), relacionando os `Employees` de cada `Company`.**

```python
queryset = Company.objects.prefetch_related('employee_set').order_by('pk')
```

## Exemplo real:

```python
queryset = Order.objects.select_related(
                                    'contract',
                                    'interest',
                                    'company',
                                    'owner',
                                ) \
                                .prefetch_related(
                                    'commitments__items__item',
                                    'contract__items',
                                ) \
                                .order_by('pk')
```

**Na entidade `Order` estamos realizando o *JOIN* com suas chaves estrangeiras `[’contract’, ‘interest’, ‘company’, ‘owner’]`, e um `prefetch_related()` com diversas instâncias acessadas através dos lookups `‘__’` de campos do Django.** 

## Evitando loops e usando métodos como `aggregate()` e `update(`)

Podemos evitar a construção de loops utilizando algumas alternativas, dependendo do contexto.

### `aggregate()`

**Insere um campo no *queryset* e retorna por fim um dicionário `(dict)`.**

### Exemplo:

- Utilizando `loop`
    
    ```python
    price = 0
    items = Item.objects.all()
    
    for item in items:
        total_price += item.price
    ```
    
- Utilizando `aggregate()`
    
    ```python
    from django.db.models import Sum
    
    items = Item.objects.all().aggregate(total_price=Sum('price'))
    
    total_price = items.get('total_price', 0)
    ```
    

### Ambas as formas estão iterando sobre os objetos da entidade Item e somando o seu campo price, porém utilizando o `aggregate` estamos realizando uma operação mais performática devido a vários motivos do Django e também de otimizações do banco de dados.

### Exemplo real:

```python
# serializers.py

price_commitment = serializers.SerializerMethodField()

def get_price_commitment(self, obj):
        price = OrderCommitmentItem.objects \
                                    .filter(
                                        item__contract=obj.contract,
                                        item__type__in=[
                                            'unit',
                                            'compound'
                                        ]
                                    ) \
                                    .aggregate(
                                        total_price=Sum(
                                            F('item__price') * F('quantity')
                                        )
                                    ) \
                                    .get('total_price', 0)

        return price
```

**Aqui estamos filtrando um conjunto de objetos, logo após estamos agregando no novo campo `total_price` os valores do campo `price` que é presente no objeto da chave primária  do campo `item`. Realizamos a agregação com o operador `Sum()` e o operador `F()`, o `F()` converte um campo para ser utilizado em operações.**

### `update()`

### Da mesma forma que o `aggregate` é mais performático do que os loops, realizar algumas alterações utilizando o `update()` se torna mais performático do que alterar na instância do objeto.

### Exemplo:

- Acessando o campo

```python
items = Items.objects.all()

my_item = items.get(pk=15)
my_item.name = 'Teste'
my_item.save()
```

- Utilizando `update()`

```python
items = Items.objects.all()

items.filter(pk=15).update(name='Teste')
```

### Ambas as maneiras estão atualizando o campo `name` do item com `pk=15`, porém a segunda se torna mais performática para atualizações em grande escala ou que não precisam acessar diretamente o campo. Já o primeiro modo é mais necessário quando precisamos performar algum cálculo ou lógica.

### Criando um `serializer` para cada contexto específico

**Caso uma `view` não vá utilizar todos os campos de um modelo ou você precise *serializar* um objeto dentro de outro `serializer`,  e esses campos demandam algum esforço para serem *serializados*, vale a pena criar um `serializer` específico do modelo para a `view`, utilizando apenas o que você vai precisar.**

### Exemplo:

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

### `Product` contém vários serializers com objetivos claramente definidos, é perceptível que o último `(ComplexProductSerializer)` gera um alto volume de dados, tornando-o menos performático, então em uma situação onde eu preciso apenas do básico, posso utilizar algum existente ou criar um novo com os campos necessários.