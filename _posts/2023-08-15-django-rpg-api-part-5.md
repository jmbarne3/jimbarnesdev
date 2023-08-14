---
title: Creating an RPG API in Python Django (Part 5) - More Stats and Equipment
description: We finally develop our primary stats and build a simple inventory and equipment system.
date: 2023-08-15 06:00:00 -0400
categories: [Development, Python]
tags: [python, django, database, rest api, postgresql]
img_path: /assets/images/posts/django-rpg-api-part-5/
image:
  path: hero.jpg
  alt:
math: true
---

In our [last post]({% post_url 2023-08-01-django-rpg-api-part-4 %}), we added jobs to our application, allowing us to control the growth rate of our character's stats based on which job they'd chosen. This week we'll implement those stats and build out a simple inventory and equipment system that allows our character's to be further customized.

> If you'd like to follow along, you can find the [complete source code from last week on GitHub](https://github.com/jmbarne3/rpgapi/tree/v0.4.0). Clone down the repository and follow the setup instructions in the readme.
{: .prompt-info }

## Goals

We'll be covering a lot this week, so let's outline what we want to accomplish.

1. Stats
    - Develop Core Stats (Strength, Dexterity, etc.)
    - Develop Calculated Stats (Attack, Defense, etc.)
    - Ensure stats are determined by assigned job and level
2. Equipment
    - Develop a simple equipment system for characters
    - Ensure equipped items properly affect stats

With our goals outlined, let's jump right in and create stats for our characters.

## Stats

Since we plan to have all of our stats are ultimately affected by the character's job and equipment, we could create them as cached, dynamic fields the way we did with the `level` field back in part 3 of this series. However, for the sake of expediency, let's just create all of our stats as simple properties, which will be computed every time their accessed. The calculation for each stat will be relatively inexpensive and we can exclude those properties from the character list view - meaning they will only be calculated for a single character at a time in the character detail view - so I think the performance hit is acceptable here.

Also for simplicity's sake, we'll use a linear equation to calculate the stats. If you recall, our jobs allow growth based on preset constants, ranging from a growth rate of 2.0 down to a growth rate of 1.0. We'll insert that variable into the following formula to calculate our stats:

$$ stat = (5 * growthRate) + (level * growthRate) $$

Let's concentrate on a single stat, say strength, and see what values we end up with using this formula. First let's consider a level 1 character with an A "rating" (or growth rate) for strength. Then see what his strength would be at level 10, 20, 50 then 99.

```python
>>> (5 * 2.0) + (1 * 2.0)
12.0
>>> (5 * 2.0) + (10 * 2.0)
30.0
>>> (5 * 2.0) + (20 * 2.0)
50.0
>>> (5 * 2.0) + (50 * 2.0)
110.0
>>> (5 * 2.0) + (99 * 2.0)
208.0
```

Our character would start out with their strength at 12 and max out - without boosts from equipment - at 208. If we consider many older games had a cap at 255 - due to the limitation of using an 8 bit integer - this seems like a pretty good place to land as an upper bound for natural stat growth. Now let's consider the growth on the low end, a character with a strength rating of F.

```python
>>> (5 * 1.0) + (1 * 1.0)
6.0
>>> (5 * 1.0) + (10 * 1.0)
15.0
>>> (5 * 1.0) + (20 * 1.0)
25.0
>>> (5 * 1.0) + (50 * 1.0)
55.0
>>> (5 * 1.0) + (99 * 1.0)
104.0
```

As expected - since our growth rate is exactly half of rating A - our stats are all half of what they would be with an A rating. One thing to note about this system is every character would see _some_ growth in all of their stats at every level. If we didn't want that, we might use rates that range between a number that is less than 1 on the low end and 1 or greater at the high end. Let's implement this function in our character model as a helper function first.

```python
class Character(models.Model):
...
    def calculate_stat(self, growth_rate) -> int:
        return math.floor(
            (5 * growth_rate) + (self.level * growth_rate)
        )
```
{: file="characters/models.py" }

### Base Stat Properties

We can now implement a property for each of our stats, utilizing this helper function to actually calculate the stat.

```python
class Character(models.Model):
...
    def calculate_stat(self, growth_rate) -> int:
        return math.floor(
            (5 * growth_rate) + (self.level * growth_rate)
        )

    @property
    def strength(self):
        return self.calculate_stat(self.job.strength_mod)
    
    @property
    def dexterity(self):
        return self.calculate_stat(self.job.dexterity_mod)
    
    @property
    def agility(self):
        return self.calculate_stat(self.job.agility_mod)
    
    @property
    def vitality(self):
        return self.calculate_stat(self.job.vitality_mod)
    
    @property
    def intelligence(self):
        return self.calculate_stat(self.job.intelligence_mod)
    
    @property
    def mind(self):
        return self.calculate_stat(self.job.mind_mod)
```
{: file="characters/models.py" }

Adding these new stats to our serializer if pretty simple. I'm going to go ahead and fold all of the stats under a field named `stats` just to make things organized. To do this, we'll use a `SerializerMethodField` again and pack the stats into a dictionary.

```python
class CharacterSerializer(serializers.ModelSerializer):
    job = SimpleJobSerializer(many=False)
    stats = serializers.SerializerMethodField()

    class Meta:
        model = Character
        fields = [
            'id',
            'job',
            'name',
            'description',
            'stats',
            'experience_points',
            'level'
        ]

    def get_stats(self, obj):
        return {
            'strength': obj.strength,
            'dexterity': obj.dexterity,
            'agility': obj.agility,
            'vitality': obj.vitality,
            'intelligence': obj.intelligence,
            'mind': obj.mind
        }
```
{: file="characters/serializers.py" }

If you open up your character detail view, you should be seeing stats now. Since we did not add the stats to our `SimpleJobSerializer`, the stats won't show up in our list view, which means the values also do not have to be calculated for each character in the list. Let's go ahead and setup our calculated stats like attack and defense.

### Calculated Stat Properties

Our calculated stats are going to be affected mostly by equipped items. However, we do want some level of growth to occur naturally as the character levels up. Because attack and defense will be the most directly affected by equipment, the natural growth of those two stats will be the slowest. Things like speed and evasion will grow more rapidly, but still slowly enough that it will be important for our characters to equip items that boost those stats if those are important to the character's build. Using [the table describing our stats from last post]({% post_url 2023-08-01-django-rpg-api-part-4 %}#defining-our-stats), let's create our properties.

```python
class Character(models.Model):
...

    @property
    def attack(self):
        retval = math.floor(5 + (self.strength * self.level / 4))
        return retval

    @property
    def defense(self):
        retval = math.floor(5 + (self.vitality * self.level / 4))
        return retval

    @property
    def magic(self):
        retval = math.floor(
            2.5 + (self.intelligence * self.level / 8) + \
            2.5 + (self.mind * self.level / 8)
        )
        return retval

    @property
    def accuracy(self):
        retval = math.floor(5 + (self.dexterity * self.level / 3))
        return retval

    @property
    def evasion(self):
        retval = math.floor(5 + (self.agility * self.level / 3))
        return retval

    @property
    def speed(self):
        retval = math.floor(5 + (self.agility * self.level / 3))
        return retval
```
{: file="characters/models.py" }

> We could immediately return the result of the `math.floor()` function. However, we're going to be adding in equipment modifications to these stats in the next section, so I'm anticipating that by setting the value to a variable and then returning it.
{: .prompt-info }

I'm sure there's a lot of specific details that could be changed with this system if we were building out an actual battle system. However, for the sake of having some example properties to work with, I think this will do the trick. We can now add these into our `stats` dictionary in our serializer, and we should see them in our detail view.

```python
class CharacterSerializer(serializers.ModelSerializer):
...

    def get_stats(self, obj):
        return {
            'strength': obj.strength,
            'dexterity': obj.dexterity,
            'agility': obj.agility,
            'vitality': obj.vitality,
            'intelligence': obj.intelligence,
            'mind': obj.mind,
            'attack': obj.attack,
            'defense': obj.defense,
            'magic': obj.magic,
            'accuracy': obj.accuracy,
            'evasion': obj.evasion,
            'speed': obj.speed
        }
```

And the output of our detail view should now look something like this:

```json

{
    "id": 1,
    "job": {
        "id": 2,
        "name": "Warrior",
        "detail_view": "http://127.0.0.1:8000/api/v1/characters/jobs/2/"
    },
    "name": "Bilbo",
    "description": "A very determined hobbit!",
    "stats": {
        "strength": 14,
        "dexterity": 12,
        "agility": 11,
        "vitality": 12,
        "intelligence": 9,
        "mind": 9,
        "attack": 12,
        "defense": 11,
        "magic": 9,
        "accuracy": 13,
        "evasion": 12,
        "speed": 12
    },
    "experience_points": 50,
    "level": 2
}
```
{: file="/api/v1/characters/1/" }

Now that we have that setup, let's go ahead and create some equipment for our character's to use!

## Equipment

In a larger game, we might create a number of different kind of items: consumables, key items, weapons, armor, ammunition and so on. However, to keep things simple, we will concentrate on weapons and armor - collectively, equipment. One way we could approach this problem is to create a single model called `Equipment` and then put all the stats necessary to represent any piece of equipment into that model, using a `ForeignKey` or a `CharField` with specific choices to determine what kind of equipment each item is, i.e. a weapon or armor, or what kind of armor, etc. There are advantages to this approach, and for a simple system like the one we're developing is exactly the approach I would take.

However, since Django takes a model based approach to representing objects, we can also leverage the polymorphism available in Python to take an object-oriented approach to designing this. While it might not necessarily be the best approach in this scenario - in my opinion it brings unnecessary complexity - it does offer us a chance to explore this particular tool available within Django's ORM.

Let's start by defining what equipment we will be creating. We will want to our characters to be able to equip weapons of various sorts. We might limit which jobs can equip which kinds of weapons, so we'll need a way of identifying the _kind_ of weapon. And we will want a few different kinds of armor. We'll keep it simple and allow for head pieces (helmets, headbands, etc.), body pieces, hand pieces and feet pieces. We won't include accessories (rings, necklaces, etc.) and we'll ignore shields as well.

Weapons must boost the attack stat. Armor must boost the defense stat. They will optionally be able to boost or reduce any of the other calculated stats. For example, mage staffs might boost attack very little, but directly boost magic a lot. Light armor might boost speed and evasion, while heavy armor reduces it. Some of these details could be baked into the design of the models. However, to keep things simple we'll say that all equipment can affect the calculated stats, minus attack and defense, and that those two stats are set depending on if the equipment is a weapon or armor. We also know each piece of equipment will need a name and a description.

### Equipment Model

Now that our our requirements are sketched, let's go ahead and start creating our models. We'll start with the equipment model, which will hold common fields that both weapons and armor will use. Because we'll never directly be using the `Equipment` model, we can create it as an abstract model.

```python
class Equipment(models.Model):
    name = models.CharField(max_length=255, null=False, blank=False)
    description = models.TextField(null=True, blank=True)
    magic_mod = models.IntegerField(null=False, blank=False, default=0)
    accuracy_mod = models.IntegerField(null=False, blank=False, default=0)
    evasion_mod = models.IntegerField(null=False, blank=False, default=0)
    speed_mod = models.IntegerField(null=False, blank=False, default=0)

    def __str__(self):
        return self.name

    class Meta:
        abstract=True
```
{: file="characters/models.py" }

Because we're marking this class as being abstract in the Meta class, Django will not create a table for this model when we create the migrations and apply them. The fields present in the class will "cascade" down to inherited classes and be created in their schemas. If we had not marked the class as abstract, Django would create the tables just like any other objects. Any derived classes would then have a one-to-one relationship back to the parent table and would be able to access the fields through a join of the two tables any time an object of the derived class was accessed.

Now that we have our base class, we can create additional classes to hold our Weapon and Armor data. Our weapons will need a type, which we can then use to control which classes can equip them and not. Our armor will need both a type and a slot to distinguish a piece of armor meant for the head from one from the body, hands or feet. The weapon and armor types should definitely be their own objects with a foreign key relationship back to the piece of equipment. It's a reasonable assumption that we might want to add some additional type of weapon or armor down the road, or that we might add more jobs that need to have access to existing weapon or armor types. The armor slots, however, are unlikely to change as that would be a fairly fundamental change to the game system. It's probably a safe assumption that it won't change in the future so it doesn't _necessarily_ have to be normalized, and therefore doesn't need its own object. So, instead, we'll implement the armor slots as a choices field with a set number of choices.

```python
class WeaponType(models.Model):
    name = models.CharField(max_length=255, null=False, blank=False)
    jobs_can_equip = models.ManyToManyField(Job)

    def __str__(self):
        return self.name


class Weapon(Equipment):
    attack = models.IntegerField(null=False, blank=False, default=0)
    weapon_type = models.ForeignKey(WeaponType, on_delete=models.CASCADE)

    def __str__(self):
        return self.name


class ArmorType(models.Model):
    name = models.CharField(max_length=255, null=False, blank=False)
    jobs_can_equip = models.ManyToManyField(Job)

    def __str__(self):
        return self.name
    

class Armor(Equipment):

    class ArmorSlot(models.TextChoices):
        HEAD = 'Head'
        BODY = 'Body'
        HANDS = 'Hands'
        FEET = 'Feet'

    defense = models.IntegerField(null=False, blank=False, default=0)
    armor_slot = models.CharField(max_length=5, null=False, blank=False, choices=ArmorSlot.choices)
    armor_type = models.ForeignKey(ArmorType, on_delete=models.CASCADE)

    def __str__(self):
        return self.name
```
{: file="characters/models.py" }

You'll notice in both the `WeaponType` and `ArmorType` classes, we used a new data type, the `ManyToManyField`. Each weapon or armor can be equipped by multiple jobs, and multiple jobs can have access to each piece of equipment. While it's not obvious from the code exactly how this is accomplished in the database, if you've ever implemented a many-to-many relationship within a database schema, what Django generates should look familiar. No fields will actually be created directly on the `WeaponType` or `ArmorType` tables. Instead, a mapping table will be created that links a `WeaponType` to a `Job` and another that links an `ArmorType` to a `Job`. The schemas end up looking like this:

**Table: _weapon_type_job_**

| Column Name | Column Type |
| --- | --- |
| weapon_type_job_id | int |
| weapon_type_id | int |
| job_id | int |

**Table: _armor_type_job_**

| Column Name | Column Type |
| --- | --- |
| armor_type_job_id | int |
| armor_type_id | int |
| job_id | int |

Which object you create the `ManyToManyField` on dictates the name of the table and the name of the primary key. However, regardless of if the field was added to the `WeaponType` and `ArmorType` objects rather than creating to fields on the `Job` object, the relationship would be exactly the same. For example, we could have added fields called `weapon_types_can_be_equipped` and `armor_types_can_be_equipped` to the `Job` object and could have chosen, from the `Job` admin screen or through its editable view, which weapons and armor could be equipped. I chose to map relationship in the opposite direction because it made more sense logically to do it this way, but the end result is the same.

### Updating our Character Model

Now that we have objects, let's go ahead and give our character a place to put that equipment. We'll modify the `Character` model and add 5 new fields, one for each slot for equipment.

```python
class Character(models.Model):
    name = models.CharField(max_length=255, null=False, blank=False)
    description = models.TextField(null=True, blank=True)
    experience_points = models.IntegerField(null=False, blank=False, default=0)
    level = models.IntegerField(null=False, blank=False, default=1, editable=False)
    job = models.ForeignKey(Job, on_delete=models.CASCADE)
    
    # Equipment
    weapon = models.ForeignKey(Weapon, null=True, blank=True, on_delete=models.SET_NULL, related_name='characters_weapon_equipped')
    head = models.ForeignKey(Armor, null=True, blank=True, on_delete=models.SET_NULL, related_name='characters_head_equipped')
    hands = models.ForeignKey(Armor, null=True, blank=True, on_delete=models.SET_NULL, related_name='characters_hands_equipped')
    body = models.ForeignKey(Armor, null=True, blank=True, on_delete=models.SET_NULL, related_name='characters_body_equipped')
    feet = models.ForeignKey(Armor, null=True, blank=True, on_delete=models.SET_NULL, related_name='characters_feet_equipped')
```
{: file="characters/models.py" }

We want it to be possible for our character to have nothing equipped in each of the slots. We could do this by providing an "Empty" object within the database the same way we did by creating the "Freelancer" job for when there is no job assigned to a character. However, in this case I think it might be a better approach to go ahead and allow these columns to be null. From a data structure perspective, our Character _must_ have a job, so it doesn't make sense for that value ever to be null. However, our characters don't necessarily have to have a piece of equipment equipped to every slot, and representing that state as null is appropriate. In the same way, if a piece of equipment we have equipped to a character suddenly disappears from the database, i.e. is deleted, then it no longer exists within out system and our character can't possibly have it equipped. The character, however, does still exist, so we don't want to delete the character, which is what would happen if we set the `on_delete` to `models.CASCADE`. Instead, by using `models.SET_NULL`, we're instructing Django to make sure the value of this nullable foreign key to be set to null if the thing it's pointing to is deleted.

In addition, it's important that we explicitly set the `related_name` parameter on the armor foreign keys. By default, Django will create a `reverse` related name for each foreign key. In the same way we can see which piece of armor our `character` has equipped in the head slot by accessing `character.head`, given that same piece of `armor`, we could see all character's that have it equipped using `armor.characters_head_equipped`. Had we not set the `related_name` manually, Django would have created one like named `armor.characters` for armor foreign key we created. This creates a conflict, as we can't have four fields named the same thing. By explicitly setting the `related_name` for each field, we avoid this conflict.

At this point, we can go ahead and create a migration and apply it. Before we can update our properties to test them out, we'll need to create some equipment for our characters to equip. As before, feel free to create your own, but if you'd rather use the example data I'm showing here, you can download [an export of my data](https://raw.githubusercontent.com/jmbarne3/rpgapi/v0.5.0/characters/fixtures/example-data.json) and apply it using the `loaddata` command.

> To load the data, you can using the command `python manage.py loaddata <path-to-example-data>`. You can also store data in a folder named "fixtures" within each application. Django automatically looks in these folders for fixture data and allows you to load them using the name of the fixture instead of the full path. For example, I will be committing a file called `example-data.json` within the fixtures folder, so you could load the data using `python manage.py loaddata example-data`.
{: .prompt-info }

Once you've created your equipment, you'll want to make sure to assign some equipment to one of your characters for further testing. Keep in mind that we have not implemented any validation yet at this point, so it will be possible to assign a head piece to the feet or a body piece to the head slot, for example. There's really no reason we can't do this at this point in things - the stats will still calculate correctly - but if you like to have internal consistency in your example data, assign appropriate pieces of equipment to each slot.

### Serializers and Admin Screens

If we're going to work with any of this data, we need to be able to create it and access it, so let's go ahead and get it added to our serializers and register the admin screens. Let's start by modifying our `admin.py`.

```python
from django.contrib import admin
from .models import (
    Character,
    Job,
    WeaponType,
    Weapon,
    ArmorType,
    Armor
)

# Register your models here.
@admin.register(Job)
class JobAdmin(admin.ModelAdmin):
    pass

@admin.register(WeaponType)
class WeaponTypeAdmin(admin.ModelAdmin):
    pass

@admin.register(Weapon)
class WeaponAdmin(admin.ModelAdmin):
    pass

@admin.register(ArmorType)
class ArmorTypeAdmin(admin.ModelAdmin):
    pass

@admin.register(Armor)
class ArmorAdmin(admin.ModelAdmin):
    pass

@admin.register(Character)
class CharacterAdmin(admin.ModelAdmin):
    pass

```
{: file="characters/admin.py" }

For our serializers, we'll need create one for weapons and one for armor. We'll likely want to create separate serializers for a list view (or when it's being displayed on a character) vs the detail view for that weapon. We'll take a very similar approach to what we did with the job serializers.

```python
from rest_framework import serializers
from rest_framework.reverse import reverse

from .models import (
    Character,
    Job,
    Weapon,
    Armor
)

class SimpleJobSerializer(serializers.ModelSerializer):
    detail_view = serializers.SerializerMethodField()

    class Meta:
        model = Job
        fields = [
            'id',
            'name',
            'detail_view'
        ]

    def get_detail_view(self, obj):
        return reverse(
            'api.characters.jobs.detail',
            kwargs={'id': obj.id},
            request=self.context['request']
        )

class JobSerializer(serializers.ModelSerializer):
    class Meta:
        model = Job
        fields = '__all__'


class SimpleWeaponSerializer(serializers.ModelSerializer):
    detail_view = serializers.SerializerMethodField()

    class Meta:
        model = Weapon
        fields = [
            'id',
            'name',
            'detail_view'
        ]

    def get_detail_view(self, obj):
        return reverse(
            'api.characters.weapons.detail',
            kwargs={'id': obj.id},
            request=self.context['request']
        )

class WeaponSerializer(serializers.ModelSerializer):
    weapon_type = serializers.StringRelatedField(many=False)

    class Meta:
        model = Weapon
        fields = '__all__'


class SimpleArmorSerializer(serializers.ModelSerializer):
    detail_view = serializers.SerializerMethodField()

    class Meta:
        model = Armor
        fields = [
            'id',
            'name',
            'detail_view'
        ]

    def get_detail_view(self, obj):
        return reverse(
            'api.characters.armor.detail',
            kwargs={'id': obj.id},
            request=self.context['request']
        )

class ArmorSerializer(serializers.ModelSerializer):
    armor_type = serializers.StringRelatedField(many=False)

    class Meta:
        model = Armor
        fields = '__all__'


class CharacterSerializer(serializers.ModelSerializer):
    job = SimpleJobSerializer(many=False)
    stats = serializers.SerializerMethodField()

    class Meta:
        model = Character
        fields = [
            'id',
            'job',
            'name',
            'description',
            'stats',
            'experience_points',
            'level'
        ]

    def get_stats(self, obj):
        return {
            'strength': obj.strength,
            'dexterity': obj.dexterity,
            'agility': obj.agility,
            'vitality': obj.vitality,
            'intelligence': obj.intelligence,
            'mind': obj.mind,
            'attack': obj.attack,
            'defense': obj.defense,
            'magic': obj.magic,
            'accuracy': obj.accuracy,
            'evasion': obj.evasion,
            'speed': obj.speed
        }
```
{: file="characters/serializers.py" }

We can then import and use the serializers to create our views:

```python
from rest_framework import generics
from .models import (
    Character,
    Job,
    Weapon,
    Armor
)

from .serializers import (
    CharacterSerializer,
    JobSerializer,
    SimpleWeaponSerializer,
    WeaponSerializer,
    SimpleArmorSerializer,
    ArmorSerializer
)

# Create your views here.
class JobListView(generics.ListAPIView):
    queryset = Job.objects.all()
    serializer_class = JobSerializer

class JobDetailView(generics.RetrieveAPIView):
    queryset = Job.objects.all()
    lookup_field = 'id'
    serializer_class = JobSerializer

class WeaponListView(generics.ListAPIView):
    queryset = Weapon.objects.all()
    serializer_class = SimpleWeaponSerializer

class WeaponDetailView(generics.RetrieveAPIView):
    queryset = Weapon.objects.all()
    lookup_field = 'id'
    serializer_class = WeaponSerializer

class ArmorListView(generics.ListAPIView):
    queryset = Armor.objects.all()
    serializer_class = SimpleArmorSerializer

class ArmorDetailView(generics.RetrieveAPIView):
    queryset = Armor.objects.all()
    lookup_field = 'id'
    serializer_class = ArmorSerializer

class CharacterListView(generics.ListCreateAPIView):
    queryset = Character.objects.all()
    serializer_class = CharacterSerializer

class CharacterDetailView(generics.RetrieveUpdateDestroyAPIView):
    queryset = Character.objects.all()
    lookup_field = 'id'
    serializer_class = CharacterSerializer

```
{: file="characters/views.py" }

And then we can add the views to our URL configuration.

```python
from django.urls import path

from .views import (
    CharacterListView,
    CharacterDetailView,
    JobListView,
    JobDetailView,
    WeaponListView,
    WeaponDetailView,
    ArmorListView,
    ArmorDetailView
)

urlpatterns = [
    path(r'jobs/<id>/',
        JobDetailView.as_view(),
        name='api.characters.jobs.detail'
    ),
    path(r'jobs/',
        JobListView.as_view(),
        name='api.characters.jobs.list'
    ),
    path(r'weapons/<id>/',
        WeaponDetailView.as_view(),
        name='api.characters.weapons.detail'
    ),
    path(r'weapons/',
        WeaponListView.as_view(),
        name='api.characters.weapons.list'
    ),
    path(r'armor/<id>/',
        ArmorDetailView.as_view(),
        name='api.characters.armor.detail'
    ),
    path(r'armor/',
        ArmorListView.as_view(),
        name='api.characters.armor.list'
    ),
    path(r'<id>/',
        CharacterDetailView.as_view(),
        name='api.characters.detail'
    ),
    path(r'',
        CharacterListView.as_view(),
        name='api.characters.list'
    ),
]

```
{: file="characters/urls.py" }

You should now be able to view the following API endpoints for weapons and armor:

- /api/v1/characters/armor/
- /api/v1/characters/armor/{id}
- /api/v1/characters/weapons/
- /api/v1/characters/weapons/{id}

## Bringing it All Together

Now that we have our serializers and views in place, we can go ahead and update our character serializers and views to include our equipment. We'll be reusing the `SimpleWeaponSerializer` and the `SimpleArmorSerializer` for our character view, and details about the weapon and armor can be retrieved using the detail URL.

```python
class CharacterSerializer(serializers.ModelSerializer):
    job = SimpleJobSerializer(many=False, read_only=True)
    stats = serializers.SerializerMethodField()
    weapon = SimpleWeaponSerializer(read_only=True)
    head = SimpleArmorSerializer(read_only=True)
    body = SimpleArmorSerializer(read_only=True)
    hands = SimpleArmorSerializer(read_only=True)
    feet = SimpleArmorSerializer(read_only=True)

    class Meta:
        model = Character
        fields = [
            'id',
            'job',
            'name',
            'description',
            'stats',
            'weapon',
            'head',
            'body',
            'hands',
            'feet',
            'experience_points',
            'level'
        ]
```
{: file="characters/serializers.py" }

If we load our character detail view, we will now see our equipment listed in the view. Django doesn't support writing to nested fields by default. You have to write your own `create` and `update` methods along with any validation logic yourself for each nested field. We will be doing some of this in the next post, but for now we've set all of the nested fields to be `read_only=True` so that we're still able to update fields that _are_ writable.

### Adding in Equipment Stats

The final piece of the puzzle for this week is to go ahead and get the stats from our equipment added to our properties. This also gives us an excuse to go over list comprehensions, a fairly commonly used syntactic device in Python, so let's take a look.

For our attack property, we're only getting a boost in stats from our `weapon` field so we can just add that in without any further modification to the rest of our model.

```python
    @property
    def attack(self):
        retval = math.floor(5 + (self.strength * self.level / 4))
        retval += self.weapon.attack if self.weapon is not None else 0
        return retval
```
{: file="characters/models.py" }

We want to ensure a weapon is equipped before trying to access a property on it. If a weapon hasn't been assigned, the value of `self.weapon` will be `None`, so we can check for this using a **conditional expression**. Conditional expressions can be expressed as "this value if this condition else this other value." So in our example, we'll be adding `self.weapon.attack` _if_ `self.weapon` is not `None`. If it is `None`, we'll add 0. We'll be using a variation on this for our other stats as all the other pieces of equipment are also nullable fields.

For our defense value, we can get additional defense stats from multiple pieces of equipment. We could handle this by listing out each piece of equipment.

```python
    @property
    def defense(self):
        retval = math.floor(5 + (self.vitality * self.level / 4))
        retval += self.head.defense if self.head is not None else 0
        retval += self.body.defense if self.body is not None else 0
        retval += self.hands.defense if self.hands is not None else 0
        retval += self.feet.defense if self.feet is not None else 0
        return retval
```
{: file="characters/models.py" }

However, since we're essentially performing the same operation 4 times, just to four different objects, this seems like an area where a for-loop might come in handy. So, we could rewrite the property as follows:

```python
    @property
    def defense(self):
        retval = math.floor(5 + (self.vitality * self.level / 4))
        armor = [self.head, self.body, self.hands, self.feet]
        for a in armor:
            retval += a if a is not None else 0
        return retval
```
{: file="characters/models.py" }

This definitely helps improve the readability and reduces the amount of code we have to repeat. You may have already noticed, however, that by defining the `armor` array inside of this property, we're not going to be able to use it in any of our other properties we need to modify. So we can move that out into its own property to make it reusable and rewrite our defense property to take advantage of it.

```python
    @property
    def armor(self):
        return [
            self.head,
            self.body,
            self.hands,
            self.feet
        ]

    ...

    @property
    def defense(self):
        retval = math.floor(5 + (self.vitality * self.level / 4))
        for a in self.armor:
            retval += a.defense if a is not None else 0
        return retval
```
{: file="characters/models.py" }

### List Comprehensions

We can further reduce this down using a handy function and a very useful feature found in Python. The first is what's called a list comprehension. List comprehensions allow you to traverse an iterable and extract out a new list. You can use it to filter, modify or format a list or many other simple tasks. If the logic you need to perform your task has several conditions or modifications, a for-loop is probably your best bet. However, if you have a simple comparison (in the case of filtering) or modification (in the case of modifying the output values) to make, a list comprehension can be very useful.

In this case, we want to traverse all the pieces of armor that are not none (our condition) and return their defense values (our modification). We will not be returning an `Armor` object in our comprehension, but rather an integer representing the defense value. We could modify our for loop to look like the following:

```python
    @property
    def defense(self):
        retval = math.floor(5 + (self.vitality * self.level / 4))
        for defense_value in [a.defense for a in self.armor if a is not None]:
            retval += defense_value
        return retval
```
{: file="characters/models.py" }

Let's concentrate on the comprehension first. The first section - `a.defense` - is our return value. Each item in the list that is returned will correspond with the value of the `defense` field on the object `a`, which is an `Armor` object. The second section - `for a in self.armor` - represents the collection we're working on. It should look familiar because the syntax is almost exactly that of a for-loop. The third section - `if a is not None` - is our conditional. The return value - `a.defense` - is only added to the returned list if the condition is met, in other words, if `a is not None` is `True`.

Because a list comprehension returns us a new list, our function above still needs to loop through the integers returned in the list and add them to our return value for the property. We can remove the need of this extra for-loop by using our second handy tool, the `sum` function. The `sum` function takes an iterable, usually a list, and adds all of the values together. If can take a list of integers, a list of a floats or a list of mixed types. If you pass it two types that cannot be added together, for example a list and an integer, it will throw an exception for the resulting unsupported operand type. Since we're getting an iterable of integers back from our list comprehension, we can use the `sum` function to add them all up, and we will have the total defensive stat for all of our armor in a single line.

```python
    @property
    def defense(self):
        retval = math.floor(5 + (self.vitality * self.level / 4))
        retval += sum([x.defense for x in self.armor if x is not None])
        return retval
```
{: file="characters/models.py" }

We now have a single line statement that goes through all the non-null pieces of armor, adds up their defensive stats and adds them to our base defense. We can use a similar approach with the rest of the stats. Because weapons and armor can both affect magic, accuracy, evasion and speed, we'll want to create one more property that return the weapon _and_ all the armor. We can simplify this property by just adding our weapon to the armor property through Python's built-in list addition operation.

```python
    @property
    def equipment(self):
        return [self.weapon] + self.armor
```
{: file="characters/models.py" }

We can then utilize this property to calculate our other stats.

```python
    @property
    def magic(self):
        retval = math.floor(
            2.5 + (self.intelligence * self.level / 8) + \
            2.5 + (self.mind * self.level / 8)
        )
        retval += sum([x.magic_mod for x in self.equipment if x is not None])
        return retval

    @property
    def accuracy(self):
        retval = math.floor(5 + (self.dexterity * self.level / 3))
        retval += sum([x.accuracy_mod for x in self.equipment if x is not None])
        return retval

    @property
    def evasion(self):
        retval = math.floor(5 + (self.agility * self.level / 3))
        retval += sum([x.evasion_mod for x in self.equipment if x is not None])
        return retval

    @property
    def speed(self):
        retval = math.floor(5 + (self.agility * self.level / 3))
        retval += sum([x.speed_mod for x in self.equipment if x is not None])
        return retval
```
{: file="characters/models.py" }

Now, if you have a character with equipment assigned, you should be able to view that stats in the detail view and see them modified by the equipment. In my local instance, I've given Bilbo the "Warrior" class and have assigned him a Bronze Sword and leather armor. This should give a boost to attack, defense and speed.

```json
{
    ...
    // Before equipment
        "stats": {
        "strength": 14,
        "dexterity": 12,
        "agility": 11,
        "vitality": 12,
        "intelligence": 9,
        "mind": 9,
        "attack": 12,
        "defense": 11,
        "magic": 9,
        "accuracy": 13,
        "evasion": 12,
        "speed": 12
    },
    ...
    // After equipment
    "stats": {
        "strength": 14,
        "dexterity": 12,
        "agility": 11,
        "vitality": 12,
        "intelligence": 9,
        "mind": 9,
        "attack": 19,
        "defense": 19,
        "magic": 9,
        "accuracy": 13,
        "evasion": 12,
        "speed": 17
    },
}
```
{: file="/api/v1/characters/1/ (Before equipment)" }

## Final Thoughts

We now have a working equipment system, but some of our validation steps are not yet in place. It is currently still possible for a character to equip a weapon or piece of armor that is for a different job or for a slot it is not intended for. Our serializers aren't able to update a character's job or equipment at all at this point. In our next post, we'll look at writing our `create` and `update` functions for characters and add in validation that enforces our rules around equipment. In addition, we'll modify our admin interface with some of Django's built-in validation and filtering functionality to ensure it enforces the same constraints.
