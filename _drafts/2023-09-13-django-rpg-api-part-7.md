---
title: Creating an RPG API in Python Django (Part 7) - Filtering
description: We create filters for our API endpoints and write a few custom filtersets.
date: 2023-11-05 06:00:00 -0400
categories: [Development, Python]
tags: [python, django, database, rest api, postgresql]
img_path: /assets/images/posts/django-rpg-api-part-7/
image:
  path: hero.jpg
  alt:
math: false
---

In our [last post]({% post_url 2023-08-28-django-rpg-api-part-6 %}), we enforced our equipment logic through the use of custom serializer fields. We were able to ensure armor could only be equipped to the correct slot and only if the armor type was allowed for the character's current job. We also implemented similar logic for weapons, restricting which weapons could be equipped to allowable weapon types per job.

If we were building out a frontend application for this API, we could limit which equipment choices were available to the user based on these restrictions by limiting the results from our armor or weapon endpoints based on their character's current job, similar to what we see happening in the HTML form fields of the API. However, in order to do this, we need a way of filtering those API endpoints, and as of yet, we have not added filters to _any_ of our endpoints! We will remedy that situation today.

> If you'd like to follow along, you can find the [complete source code from last week on GitHub](https://github.com/jmbarne3/rpgapi/tree/v0.6.0). Clone down the repository and follow the setup instructions in the readme.
{: .prompt-info }

## Goals

We want to add a few filters to each of our primary endpoints. There are a few different ways to approach filtering with Django Rest Framework, some of which are available in the framework itself, and others that require the additional [`django-filter` library](https://pypi.org/project/django-filter/). Let's add the library `django-filter` to our dependency list in the `pyproject.toml` and run `pip install .` install it.

We want to be able to perform a text search against things like the character's name and description, which we can do out-of-the-box with Django Rest Framework. We'll also want to be able to do some many-to-many-through filtering with a custom query param name for filtering weapons and armor by job, which we'll need to create a custom filter to do using `django-filter`. Here's the breakdown of what we want to add to our endpoints:

* `/api/v1/characters/`
  * `search` - Searches for matches within name and description
  * `level` - Filters results by level
  * `job` - Filters results by job
* `/api/v1/characters/armor/`
  * `job` - Filters armor by equippable job
  * `type` - Filters armor by type
  * `slot` - Filters armor by slot
* `/api/v1/characters/weapons/`
  * `job` - Filters armor by equippable job
  * `type` - Filters weapon by type


## Some Preparations

We need to knock out a couple of things before we can get started. First, since we're going to be filtering sets of results, it might be helpful to have more than a handful of objects in our database. I went ahead and created a few more armor sets and added a few more characters with various jobs and levels to our [example data file, which you can download here](/assets/images/posts/django-rpg-api-part-7/example-data.json). You can import this using the `python manage.py loaddata` command, providing the app and file names as arguments: `python manage.py loaddata characters example-data.json`.

I also noticed while writing this post that our calculated stat functions were off just a bit. For example, when creating the "Gandalf" character in the example, I noticed that at around level 50, he had calculated stats up above 6-700, which is much higher than what we were shooting for. This turned out to a simple math error which was missed in an earlier post that we can pretty easily correct.

> Note, this problem would have been discovered has we written a test for the stat calculations when we wrote that logic! I decided to hold off on writing tests until the end of the series so I could cover testing in a single post, but I think this little issue demonstrates the importance of writing tests as you go, rather than waiting until the end of a sprint to implement them.
{: .prompt-info }

Our goal here is to start with a small base amount, 5, linearly boost each stat per level and then add some additional amount based on a base stat. So, for example, the attack stat will be calculated as 5 + the character's level plus 3/4 the character's strength. This results in each stat always increasing by 1 for all character classes, but classes with greater growth rates for a particular base stat might see the stat increase by 2 or even 3 when leveling up. I've rewritten the calculated stats as follows:

```python
    @property
    def attack(self):
        retval = math.floor(5 + (3 * self.strength / 4) + self.level)
        retval += self.weapon.attack if self.weapon is not None else 0
        return retval

    @property
    def defense(self):
        retval = math.floor(5 + (3 * self.vitality / 4) + self.level)
        retval += sum([x.defense for x in self.armor if x is not None])
        return retval

    @property
    def magic(self):
        retval = math.floor(
            2.5 + (3 * self.intelligence / 8) + \
            2.5 + (3 * self.mind / 8) + \
            self.level
        )
        retval += sum([x.magic_mod for x in self.equipment if x is not None])
        return retval

    @property
    def accuracy(self):
        retval = math.floor(5 + (2 * self.dexterity / 3) + self.level)
        retval += sum([x.accuracy_mod for x in self.equipment if x is not None])
        return retval

    @property
    def evasion(self):
        retval = math.floor(5 + (2 * self.agility / 3) + self.level)
        retval += sum([x.evasion_mod for x in self.equipment if x is not None])
        return retval

    @property
    def speed(self):
        retval = math.floor(5 + (2 * self.agility / 3) + self.level)
        retval += sum([x.speed_mod for x in self.equipment if x is not None])
        return retval
```
{: file="characters/models.py" }

This results in a much more balanced set of calculated stats in the higher levels than what we were seeing before:

```json
{
    "id": 4,
    "job": 6,
    "job_details": {
        "id": 6,
        "name": "Black Mage",
        "detail_view": "http://127.0.0.1:8000/api/v1/characters/jobs/6/"
    },
    "name": "Gandalf",
    "description": "A powerful wizard",
    "stats": {
        "strength": 75,
        "dexterity": 75,
        "agility": 86,
        "vitality": 75,
        "intelligence": 108,
        "mind": 97,
        "attack": 113,
        "defense": 120,
        "magic": 151,
        "accuracy": 104,
        "evasion": 112,
        "speed": 112
    },
    "weapon": 5,
    "weapon_details": {
        "name": "Ash Wand",
        "detail_view": "http://127.0.0.1:8000/api/v1/characters/weapons/5/"
    },
    "head": 9,
    "head_details": {
        "name": "Ribbon",
        "detail_view": "http://127.0.0.1:8000/api/v1/characters/armor/9/"
    },
    "body": 10,
    "body_details": {
        "name": "Black Robe",
        "detail_view": "http://127.0.0.1:8000/api/v1/characters/armor/10/"
    },
    "hands": 12,
    "hands_details": {
        "name": "Copper Armlet",
        "detail_view": "http://127.0.0.1:8000/api/v1/characters/armor/12/"
    },
    "feet": 13,
    "feet_details": {
        "name": "Black Boots",
        "detail_view": "http://127.0.0.1:8000/api/v1/characters/armor/13/"
    },
    "experience_points": 25000,
    "level": 49
}
```

And with that we can go ahead and move forward with our endpoint filters!

## Character Filterset

We'll start out with our character endpoint. We'll be able to cover most of the important concepts surrounding filtering with this endpoint, so it's a good place to start. Each filter will require a different approach, the `search` filter more of less working out of the box, the `level` filter being provided by the `django-filter` module, and the `job` requiring a little bit of customization.

First, for any custom filters, we'll want to store them in a separate file, so go ahead and create a file named `filters.py` in the `characters` directory. We can leave this empty for now, but we'll be coming back to it very soon. While we will be creating any custom filters here, all filters are assigned to a particular view. So we will need to import any filters we want to use inout our `views.py` file.

To create our search filter, we're going to need to import the `SearchFilter` class from the rest_framework module into our`views.py` file. Since we will be using multiple filters, it might be simpler t import the `filters` module so we have access to all of them:

```python
from rest_framework import generics, filters

from .models import (
    Character,
    Job,
    Weapon,
    Armor
)
...
```
{: file="characters/views.py" }

### Search Filter

Once we have this imported, adding it to a view is relatively simple. The `SearchFilter` is a type of "filter backend." Filter backends have parameters that can be adjusted to change the behavior of that backend. The `SearchFilter` is going to allow us to search one or more fields and return results that include the searched text. To use it, we will need to add it to the `filter_backends` array and then configure its `search_fields` property. We do both of these directly in the class declaration.

```python
class CharacterListView(generics.ListCreateAPIView):
    queryset = Character.objects.all()
    serializer_class = SimpleCharacterSerializer
    
    filter_backends = [
        filters.SearchFilter,
    ]
    
    search_fields = ['name', 'description']
```
{: file="characters/views.py" }

For our search fields, I've chosen the player's name and description. You could add additional fields to this array. For example, if you want to be able to search by job name, you can add `job__name` as one of the search fields. All lookups are case insensitive, and all matches of any field will be included (i.e., if the search query is part of the name, description _or_ job name fields). We can down use the `?search=` query parameter on our endpoint, and Django will perform the filtering for us.

### Level Filter

Let's go ahead and add another simple filter. In this case, we're going to filter results based on the provided level. As a starting point, we'll only return results that match a _specific_ level. First, we'll need to add the `DjangoFilterBackend` from the `django_filters.rest_framework.backends` module.

```python
from rest_framework import generics, filters
from django_filters.rest_framework.backends import DjangoFilterBackend
from .models import (
    Character,
    Job,
    Weapon,
    Armor
)
```
{: file="characters/views.py" }

We can then add it to our list of filter backends on the `CharacterListView` class and configure a filter field.

```python
class CharacterListView(generics.ListCreateAPIView):
    queryset = Character.objects.all()
    serializer_class = SimpleCharacterSerializer
    
    filter_backends = [
        DjangoFilterBackend,
        filters.SearchFilter,
    ]
    
    filterset_fields = ['level']
    search_fields = ['name', 'description']
```
{: file="characters/views.py" }

In our example data, if we search for character's with a level of 2, we should get a single record back, our old friend Bilbo. Check this and ensure everything is working.

```json
[
    {
        "id": 1,
        "job": 4,
        "name": "Bilbo",
        "description": "A very determined hobbit!",
        "stats": {
            "strength": 11,
            "dexterity": 12,
            "agility": 14,
            "vitality": 11,
            "intelligence": 11,
            "mind": 9,
            "attack": 19,
            "defense": 23,
            "magic": 14,
            "accuracy": 15,
            "evasion": 16,
            "speed": 23
        },
        "experience_points": 50,
        "level": 2
    }
]
```
{: '/api/v1/characters/?level=2" }

Although the interface in the HTML form is not the best (you might note the search field and our filterset fields each have their own submit button), it is possible to use both fields simultaneously. Try searching for `/api/v1/characters/?search=Bilbo&level=2` and you should still see our record be Bilbo returned. Try changing one parameter and then the other to some invalid value to ensure they're both working together to return that record.

### Filter by Job

Now, we'd like to go ahead and implement our job filter. This is also an extremely simple addition, as adding the `job` field to the `fieldset_fields` will take care of all the work for us:

```python
class CharacterListView(generics.ListCreateAPIView):
    queryset = Character.objects.all()
    serializer_class = SimpleCharacterSerializer
    
    filter_backends = [
        DjangoFilterBackend,
        filters.SearchFilter,
    ]
    
    filterset_fields = ['level', 'job']
    search_fields = ['name', 'description']
```
{: file="characters/views.py" }

We should now be able to filter our results by job by providing the job ID as a filter: `/api/v1/characters/?job=2`. And with that we've met our goals for this particular endpoint. However, being able to filter for a particular level doesn't seem particularly useful to me, and I'd really like to have a nice clean form that includes the search field _and_ our level and job filters so we can apply all three at once from the GUI. And since we'll be moving in a similar direction for our armor and weapon filters, I think it's time to jump into creating a custom filterset.

### Custom Filtersets

The `django-filter` package allows us to package up all the filters we might want to use in a view in one class and then just apply that class to the view. For our particular scenario there is one downside to this approach: these filtersets are made to encapsulate functionality from the `django-filter` package, so we won't be able to use the `SearchFilter` functionality that comes out-of-the-box from Django Rest Framework. Fortunately, it's fairly easy to get the same results from a custom method filter in the `django-filter` package.

We'll start out by creating a custom filterset in our `filters.py` file for the character view.

```python
from django_filters import FilterSet
from .models import Character

class CharacterFilterSet(FilterSet):
    class Meta:
        model = Character
        fields = ['level', 'job']
```
{: file="characters/filters.py" }

We've created a new class that is derived from the `django_filters.FilterSet` class and within the inner `Meta` class, we've set the model to `Character` and added the `level` and `job` fields. You'll note the `search` field isn't here yet. We'll need to define that as a custom field first before we can add it. In order to use the filterset, we need to import it into our `views.py` file and add it to the `filterset_class` property.

```python
from .filters import CharacterFilterSet

...

class CharacterListView(generics.ListCreateAPIView):
    queryset = Character.objects.all()
    serializer_class = SimpleCharacterSerializer
    filterset_class = CharacterFilterSet
```
{: file="characters/views.py" }

Since we'll be using the `DjangoFilterBackend` as the default backend for the rest of the models, we can go ahead and set that as the default within our `settings.py` which will allow us to remove it from our class definitions.

```python
REST_FRAMEWORK = {
    'DEFAULT_FILTER_BACKENDS': (
        'django_filters.rest_framework.DjangoFilterBackend',
        ...
    ),
}
```
{: file="rpgapi/settings.py" }

> Note, you must add this as the default in the `settings.py` for the examples I'm showing to work. If it is not defined, you will need to add back in the `filter_backends` array to each view.
{: .prompt-info }

If you refresh your character list view, you should still have the filters option on the screen and see the level and job filters. Now, let's go ahead and create our search field. To do this, we're going to create a `CharFilter` and then override the method it calls. This allows us to modify the queryset based on the value passed to the custom filter.

```python
from django_filters import FilterSet, CharFilter
from .models import Character

from django.db.models import Q

class CharacterFilterSet(FilterSet):
    search = CharFilter(label='Search', method='custom_search')

    class Meta:
        model = Character
        fields = ['level', 'job']

    def custom_search(self, queryset, name, value):
        return queryset.filter(
            Q(name__icontains=value) |
            Q(description__icontains=value)
        )
```
{: file="characters/filters.py" }

You'll notice a few things in our code above. First, we define our `search` filter. Since the field `search` doesn't correspond with a field within our `Character` model, we need to provide both a label, which will determine what label is used on the HTML form, and a method parameter. The method parameter will tell Django what to do with the value passed to the filter. We named our method `custom_search` and defined the method below the Meta class definition.

The second thing you may notice is the use of the `Q()` function within our `custom_search` method. We haven't yet come across this in our posts yet, but it is a very useful way to encapsulate part of a filter clause, especially when you need to perform an `OR` operation on the where clause. By default, when you add filters to a Django queryset, each additional filter is treated like an `AND` statement onto whatever existing clauses are in the `WHERE` statement. So for example, let's assume we've queried our characters and done an initial filter that looks something like this: `characters = Character.objects.filter(job=2)`. This would generate a SQL statement that looks something like the following:

```sql
SELECT
    *
FROM
    characters_character c
WHERE
    c.job_id = 2;
```
{: file="SQL Statement" }

If we were then to additionally filter for level, we might do something like: `characters = characters.filter(level__gte=2)`. This would add on an additional clause in the `WHERE` statement:

```sql
SELECT
    *
FROM
    characters_character c
WHERE
    c.job_id = 2
AND
    level >= 2;
```

However, what if we didn't want to add that filter in as an `AND` but rather as an `OR`? The easiest way to do this is to `OR` two separate filters together. We can do this using `Q()` functions, placing our filter logic within each `Q()` function. So to combine the two statements as an `OR` we could use the following: `characters = Character.objects.filter(Q(job=2)|Q(level__gte=2))`. This would generate a SQL statement like the following:

```sql
SELECT
    *
FROM
    characters_character c
WHERE
    c.job_id = 2
OR
    level >= 2;
```

This is exactly what we're doing in our `custom_search` function. We want any records where the name field contains the search value, case insensitive, _or_ the description field contains the search value, case insensitive. 

The last thing you may notice is the search field isn't in a list of fields. This is because the _only_ fields that should be defined in this array are the model fields we need Django to automatically setup for us. We've manually defined our search field, so it should not be added. This is also true if you override a model field to change its behavior. Defining the field within the `fields` array will cause it to automatically setup the field, effectively overriding any customizations you've made.

We can add one last bit of customization to our FilterSet by providing a "range" capability for our level filter. This can be very easily accomplished by updating the lookup expressions available to the field.

```python
class CharacterFilterSet(FilterSet):
    search = CharFilter(label='Search', method='custom_search')

    class Meta:
        model = Character
        fields = {
            'level': ['exact', 'range'],
            'job': ['exact']
        }

    def custom_search(self, queryset, name, value):
        return queryset.filter(
            Q(name__icontains=value) |
            Q(description__icontains=value)
        )
```
{: file="characters/filters.py" }

We've updated our `fields` field from an array to an object, with each field defined as a key and value. The key is the field name, in this case "level" and "job", and the value can be a few things, but in this case is the array of lookup expressions we want to make available. For the `level` field, I've included the default "exact" lookup, which looks up an exact match of the value passed in, along with the "range" lookup expression. This allows for a comma delimited range of values to be passed, which it will then use to build an expression that pulls records which values are greater than some minimum you pass in and less than some maximum. You can test this by refreshing the character list view and adding `?level__range=1,30` as a query parameter. If you're using my example data, you should get all the characters back except for Gandalf, who is level 49 and outside of the specified range.

And with that, I think we've now finished up our character filters and can move onto our armor and weapon filters.

## Armor Filterset

Now that we've had some practice building custom filters, these next two should be fairly straightforward. Let's start by outlining our FilterSet class and applying it to our armor list view. Be sure to add the appropriate imports for your `Armor` class at the top of the `filters.py` file.

```python
class ArmorFilterSet(FilterSet):
    class Meta:
        model = Armor
        fields = ['armor_slot', 'armor_type']
```
{: file="characters/filters.py" }

This will allow us to filter on the slot and type of the armor returned in the list. To get this initial set of filters added to our view, we need to import it and set it as the `filterset_class` on the `ArmorListView`.

```python
from .filters import ArmorFilterSet, CharacterFilterSet

...

class ArmorListView(generics.ListAPIView):
    queryset = Armor.objects.all()
    serializer_class = SimpleArmorSerializer
    filterset_class = ArmorFilterSet
```
{: file="characters/views.py" }

We should see the filter option at the top of the view and, when we click on it, see the "Armor slot" and "Armor type" filters.

![Image of the armor filterset within the HTML API View](armor_filter_set_initial.png)

