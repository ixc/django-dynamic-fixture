.. _ddf:

Main features
*******************************************************************************

Get: G
===============================================================================

The **G** function is the main feature of DDF. It receives a model class and it will return a valid and persisted instance filled with dynamically generated data::


    from django_dynamic_fixture import G, get
    instance = G(MyModel)
    # The same as:
    # instance = get(MyModel)
    print instance.id # this will print the ID, indicating the instance was saved
    print instance.some_field # this will print the auto generated data


This facilitates writing tests and it hides all dummy data that polutes the source code. But all important data of the test must be explicitily defined::


    instance = G(MyModel, my_field=123, another_field='abc')
    print instance.my_field # this will print 123
    print instance.another_field # this will print 'abc'


Important details:

* The id (AutoField) is auto filled, unless you set a value to it.
* if a field has default value, it is used by default.
* if a field has choices, first option is selected by default.


New: N
===============================================================================

This function **N** is similar to **G**, except it will not save the generated instance, only all internal dependencies of *ForeignKey* and *OneToOneField* fields. Since the instance does not have an ID, it can not insert instances in *ManyToManyField* fields. This function may be useful mainly in one of the following situations: create a unit test independent of the database; or
create a not persisted instance that will be manipulated before saving it (usually dealing with custom fields, custom validations etc)::

    from django_dynamic_fixture import N, new
    instance = N(MyModel)
    # The same as:
    # instance = new(MyModel)
    print instance.id # this will print None
    print instance.some_field # this will print the auto generated data


It is possible to disable saving all instances, but it has to be disabled manually::

    instance = N(MyModel, persist_dependencies=False)
    print instance.id # this will print None
    print instance.some_fk_field.id # this will print None


Fixture: F
===============================================================================

It is possible to explicitly set a value for a relationship field(*ForeingKey*, *OneToOneField* and *ManyToManyField*) using the **F** function::

    from django_dynamic_fixture import G, F, fixture
    instance = G(MyModel, my_fk_field=F(my_field=1000))
    # The same as:
    # instance = G(MyModel, my_fk_field=fixture(my_field=1000))
    print instance.my_fk_field.my_field # this will print 1000

This is the equivalent to::

    my_fk_field = G(MyOtherModel, my_field=1000)
    instance = G(MyModel, my_fk_field=my_fk_field)
    print instance.my_fk_field.my_field # this will print 1000

**F** function is recursible::

    instance = G(MyModel, fk_a=F(fk_b=F(field_c=1000)))
    print instance.fk_a.fk_b.field_c # this will print 1000

**F** can be used to customize instances of a ManyToManyField too::

    instance = G(MyModel, many_to_many_field=[F(field_x=1), F(field_x=2)])
    print instance.many_to_many_field.all()[0].field_x # this will print 1
    print instance.many_to_many_field.all()[1].field_x # this will print 2


Many to Many fields
===============================================================================

DDF can add instances in *ManyToManyFields*, but only if the instance has been persisted to the database (Django requirement). DDF handles *ManyToManyFields* with *through* table as well.

It is possible to define how many instances will be created dynamically::

    instance = G(MyModel, many_to_many_field=5)
    print instance.many_to_many_field.all().count() # this will print 5

It is possible to customize each instance of the *ManyToManyField*::

    instance = G(MyModel, many_to_many_field=[F(field_x=10), F(field_y=20)])
    print instance.many_to_many_field.all().count() # this will print 2
    print instance.many_to_many_field.all()[0].field_x # this will print 10
    print instance.many_to_many_field.all()[1].field_y # this will print 20

It is possible to pass already created instances too::

    instance1 = G(MyRelatedModel)
    instance2 = G(MyRelatedModel)

    instance = G(MyModel, many_to_many_field=[instance1, instance2])
    print instance.many_to_many_field.all().count() # this will print 2

    instance = G(MyModel, many_to_many_field=[F(), instance1, F(), instance2])
    print instance.many_to_many_field.all().count() # this will print 4


Django Look Up fields syntax (New in 1.6.1)
===============================================================================

This is an alias to F function, but it follows the Django pattern of filters that use two underlines to access internal fields of foreign keys::

    from django_dynamic_fixture import G
    instance = G(MyModel, myfkfield__myfield=1000)
    print instance.myfkfield__myfield # this will print 1000

Just be careful because DDF does not interpret related names yet.


Global Settings
===============================================================================

You can configure DDF in ``settings.py`` file. You can also override the global config per instance creation when necessary.

* **DDF_FILL_NULLABLE_FIELDS** (Default = True): DDF can fill nullable fields (``null=True``) with None or some data::

    # SomeModel(models.Model): nullable_field = Field(null=True)
    G(SomeModel).nullable_field is None # True if DDF_FILL_NULLABLE_FIELDS is True
    G(SomeModel).nullable_field is None # False if DDF_FILL_NULLABLE_FIELDS is False

    # You can override the global config for one case:
    G(Model, fill_nullable_fields=False)
    G(Model, fill_nullable_fields=True)


*  **DDF_VALIDATE_MODELS** (Default = False): DDF will call ``model.full_clean()`` method before saving to the database::

    # You can override the global config for one case:
    G(Model, validate_models=True)
    G(Model, validate_models=False)


* **DDF_VALIDATE_ARGS** (Default = False): DDF can raise an exception if you are trying to fill data for a non existent field::

    G(Model, nonexistent_field='data') # DDF will ignore it if DDF_VALIDATE_ARGS is False
    G(Model, nonexistent_field='data') # DDF will raise an exception if DDF_VALIDATE_ARGS is True

    # You can override the global config for one case:
    G(Model, validate_args=False)
    G(Model, validate_args=True)


* **DDF_FIELD_FIXTURES** (Default = {}) (new in 1.8.0): Dictionary where the key is the full qualified name of the field and the value is a function without parameters that returns a value::

    DDF_FIELD_FIXTURES = {'path.to.your.Field': lambda: random.randint(0, 10) }


* **DDF_USE_LIBRARY** (Default = False): For using the Shelve feature::

    # You can override the global config for one case:
    G(Model, use_library=False)
    G(Model, use_library=True)


* **DDF_NUMBER_OF_LAPS** (Default = 1):  For models with foreign keys to itself (``ForeignKey('self')``), DDF will avoid infinite loops because it stops creating objects after it create **n** **laps** for the cycle::

    # You can override the global config for one case:
    G(Model, number_of_laps=5)


* **DDF_DEBUG_MODE** (Default = False): To show some DDF logs::

    # You can override the global config for one case:
    G(Model, debug_mode=True)
    G(Model, debug_mode=False)
