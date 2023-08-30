---
title: Creating an RPG API in Python Django (Part 6) - Field Validation
description: We enforce our equipment constraints by customizing our serializers.
date: 2023-08-28 06:00:00 -0400
categories: [Development, Python]
tags: [python, django, database, rest api, postgresql]
img_path: /assets/images/posts/django-rpg-api-part-6/
image:
  path: hero.jpg
  alt:
math: false
---

In our [last post]({% post_url 2023-08-14-django-rpg-api-part-5 %}), we integrated our stat and equipment system into our character class and serializers, allowing us to assign and remove equipment pieces from our characters. However, it is currently still possible for our characters to equip weapons or armor that should not be allowed due to their current job assignment or improper slots. We could account for this limitation within our model, by creating specific functions for adding and removing equipment.

However, I would argue this would be an improper place to create this logic. As far as the model is concerned, any piece of armor _should_ be able to be equipped to any armor slot. If we wanted this limitation to be represented in the model, it would be more proper to further abstract the pieces of equipment into different classes, one for each armor slot. While this approach is perfectly fine, it will add complexity to the creation of equipment within the application and does nothing to deal with our other limitation, only allowing armor and weapons to be equipped if it is indicated the character's job is allowed to equip them.

Therefore, we will place these limitations within the "controller" layer - which within the Django Rest Framework lies primarily in the serializers. This will allow the model to accurately reflect the database schema it describes while allowing for our "business logic" to be placed in the appropriate layer of the application.

> If you'd like to follow along, you can find the [complete source code from last week on GitHub](https://github.com/jmbarne3/rpgapi/tree/v0.5.0). Clone down the repository and follow the setup instructions in the readme.
{: .prompt-info }

## Goals

We have two primary goals this week, both related to one another. First, we want to ensure that only armor of a particular `armor_slot` can be equipped in each slot on our character. So, armor for your head should only be able to be equipped to the head slot, only body armor on the body, etc. Secondly, we want to ensure the `armor_type` and `weapon_type`, respectively, of a piece of equipment is allowed to be equipped to the character's current job. This is controlled by the `jobs_can_equip` fields on the `ArmorType` and `WeaponType` classes.

I should clarify that for this particular challenge, we're concerned with ensuring this behavior when changes are made through our API endpoints. As described above, we're not setting restrictions on the model itself, so it will still be possible to assign an improper weapon or piece of armor when working directly through the model, say through the shell. However, our API endpoints should prevent these assignments from being made.

We'll start off with a simple approach for handling the `armor_slot` problem, and then expand on what we learn to deal with the class-specific restrictions.

## Armor Slot Restrictions

Before we can work this problem out, I'd like to make one important change to our character serializer. In general, when creating API endpoints that have multiple uses, e.g. our character detail view which can retrieve, update, and delete, I like to make sure the schema is as identical as possible for various requests. So, for example, we currently have a nested serializer for each weapon and armor piece with multiple fields for each serializer.

```json
{
    "id": 1,
    "job": {
        "id": 4,
        "name": "Thief",
        "detail_view": "http://127.0.0.1:8000/api/v1/characters/jobs/4/"
    },
    "name": "Bilbo",
    "description": "A very determined hobbit!",
    "stats": {
        "strength": 11,
        "dexterity": 12,
        "agility": 14,
        "vitality": 11,
        "intelligence": 11,
        "mind": 9,
        "attack": 14,
        "defense": 18,
        "magic": 10,
        "accuracy": 13,
        "evasion": 14,
        "speed": 21
    },
    "weapon": {
        "id": 2,
        "name": "Bronze Dagger",
        "detail_view": "http://127.0.0.1:8000/api/v1/characters/weapons/2/"
    },
    "head": {
        "id": 1,
        "name": "Leather Cap",
        "detail_view": "http://127.0.0.1:8000/api/v1/characters/armor/1/"
    },
    "body": {
        "id": 3,
        "name": "Leather Jerkin",
        "detail_view": "http://127.0.0.1:8000/api/v1/characters/armor/3/"
    },
    "hands": {
        "id": 2,
        "name": "Leather Gloves",
        "detail_view": "http://127.0.0.1:8000/api/v1/characters/armor/2/"
    },
    "feet": {
        "id": 4,
        "name": "Leather Boots",
        "detail_view": "http://127.0.0.1:8000/api/v1/characters/armor/4/"
    },
    "experience_points": 50,
    "level": 2
}
```
{: file="/api/v1/characters/1/" }

We currently can't update equipment on our characters in this view because each of these serializers is set to be read-only. We _could_ modify the serializers to have an `update` function, which would allow us to accept incoming data and write it, but the schema we'd be expecting would be different from what we display. We would expect to receive a foreign key ID and then assign that to the field. However, what we display is an object that _contains_ the foreign key along with the name and a URL to retrieve the equipment's details. So while it is possible to make this work, I think it would be better to present each equipment field in the format we expect for a `PUT` request and then provide the details in a separate field. This makes the endpoint self-documenting. It also provides a nice auto-generated interface within the API views that Django provides out of the box.

To make this change, we'll simply replace our custom serializers with `PrimaryKeyRelatedField` serializer fields. This is the field Django would use by default for a primary key field anyway, and we're only explicitly using them here because we're going to modify their behavior in a moment.

```python
class CharacterSerializer(serializers.ModelSerializer):
    job = SimpleJobSerializer(many=False, read_only=True)
    stats = serializers.SerializerMethodField()
    weapon = serializers.PrimaryKeyRelatedField()
    head = serializers.PrimaryKeyRelatedField()
    body = serializers.PrimaryKeyRelatedField()
    hands = serializers.PrimaryKeyRelatedField()
    feet = serializers.PrimaryKeyRelatedField()
```
{: file="characters/serializers.py" }

If you save and reload your character view, you will immediately be met with an assertion error:

```console
AssertionError: Relational field must provide a `queryset` argument, override `get_queryset`, or set read_only=`True`.
```

And this provides a clue to how we're going to solve the next problem. First, however, let's go ahead and let Django do the work for us and see what our result _should_ look like when we're all done. Go ahead and comment out all the equipment fields and reload the page.

```python
class CharacterSerializer(serializers.ModelSerializer):
    job = SimpleJobSerializer(many=False, read_only=True)
    stats = serializers.SerializerMethodField()
    # weapon = serializers.PrimaryKeyRelatedField()
    # head = serializers.PrimaryKeyRelatedField()
    # body = serializers.PrimaryKeyRelatedField()
    # hands = serializers.PrimaryKeyRelatedField()
    # feet = serializers.PrimaryKeyRelatedField()
```
{: file="characters/serializers.py" }

Upon reloading the page, you should see our serializers have been replaced by the foreign key IDs of each piece of equipment, and the HTML form below now has all of the pieces of equipment available to update.

```json
{
    "id": 1,
    "job": {
        "id": 4,
        "name": "Thief",
        "detail_view": "http://127.0.0.1:8000/api/v1/characters/jobs/4/"
    },
    "name": "Bilbo",
    "description": "A very determined hobbit!",
    "stats": {
        "strength": 11,
        "dexterity": 12,
        "agility": 14,
        "vitality": 11,
        "intelligence": 11,
        "mind": 9,
        "attack": 14,
        "defense": 18,
        "magic": 10,
        "accuracy": 13,
        "evasion": 14,
        "speed": 21
    },
    "weapon": 2,
    "head": 1,
    "body": 3,
    "hands": 2,
    "feet": 4,
    "experience_points": 50,
    "level": 2
}
```
{: file="/api/v1/characters/1/" }

![The HTML form shows updatable fields for all equipment.](html_form.png)
_The HTML form shows updatable fields for all equipment._

You may notice that I've switched Bilbo's job to Thief because within my data set thieves can only equip daggers. In addition, they also can only equip light armor - which is the only armor I've created in my dataset so far - so as we're testing I ultimately expect to only find one piece of available equipment for each slot. Currently, I can see all the available weapons and armor in each slot within my view. Let's concentrate on filtering each armor slot to only accept armor of the correct `ArmorSlot`.

### Overriding the Queryset

We saw when we tried to use the `PrimaryKeyRelatedField` earlier that we must provide a queryset to the field. We can use this to tell the field which `Weapon` or `Armor` objects are allowed to be assigned to the field. In the case of restricting by `ArmorSlot`, we can do this very easily, because we can hard code into each slot which types of armor are allowed. Uncomment the field definitions for the armor in the serializer, and add a filtered dataset for the queryset parameter of each.

```python
class CharacterSerializer(serializers.ModelSerializer):
    job = SimpleJobSerializer(many=False, read_only=True)
    stats = serializers.SerializerMethodField()
    # weapon = serializers.PrimaryKeyRelatedField()
    
    head = serializers.PrimaryKeyRelatedField(
        queryset=Armor.objects.filter(armor_slot=Armor.ArmorSlot.HEAD)
    )
    
    body = serializers.PrimaryKeyRelatedField(
        queryset=Armor.objects.filter(armor_slot=Armor.ArmorSlot.BODY)
    )

    hands = serializers.PrimaryKeyRelatedField(
        queryset=Armor.objects.filter(armor_slot=Armor.ArmorSlot.HANDS)
    )

    feet = serializers.PrimaryKeyRelatedField(
        queryset=Armor.objects.filter(armor_slot=Armor.ArmorSlot.FEET)
    )
```
{: file="characters/serializers.py" }

If you now refresh your character detail view, you should only see the correct pieces of armor as the options available for the HTML form. Because the queryset available only includes `Armor` objects that match the slot we're assigning them to, not only do we get a nice UI perk of having the correct objects show up in the dropdown, but Django will automatically validate any incoming requests to ensure only objects in the provided queryset can be assigned. Try this for yourself by trying to assign a body piece (in my case a Leather Jerkin, ID 3) to the `head` field by using the "Raw data" tab:

```json
{
    "head": 3
}
```
{: file="PUT Request /api/v1/characters/1/" }

If you use the `PUT` request, you should see the following error:

```json
{
    "name": [
        "This field is required."
    ],
    "head": [
        "Invalid pk \"3\" - object does not exist."
    ],
    "body": [
        "This field is required."
    ],
    "hands": [
        "This field is required."
    ],
    "feet": [
        "This field is required."
    ]
}
```
{: file="PUT Response /api/v1/characters/1/" }

> If you used the `PATCH` request, you will only see the error for the one field you attempted to update.
{: .prompt-info }

We have successfully created validation for the armor slots! Now let's go ahead and add validation for the job type requirements.

## Job Type Requirements

So based on our success with overriding the queryset parameter, it seems like we should be able to do the same thing for restricting assignment by the job. Let's start with the `weapon` field this time since there are no slot restrictions to worry about. Once we solve the problem in isolation for that field, we can work on incorporating it into our various armor fields. We know we want to override the queryset and that we want to pull weapons, so let's start by just pulling all the weapons.

```python
    weapon = serializers.PrimaryKeyRelatedField(
      queryset=Weapon.objects.all()
    )
```
{: file="characters/serializers.py" }

Okay, wonderful! Now the field should be working exactly as it was before. Now we need to use a little fancy Django syntax magic to ensure the weapon type we're using can be equipped by our current job. We'll need to pass through a couple of layers. First, our weapon has a weapon type. Then, our weapon type has a `jobs_can_equip` field that specifies which jobs can be equipped. In Django, you can pass through primary key references using a double underscore in your filter. So, for example, let's say we were going to filter weapons down to only daggers. We could do the following:

```python
    weapon = serializers.PrimaryKeyRelatedField(
      queryset=Weapon.objects.filter(weapon_type=2)
    )
```
{: file="characters/serializers.py" }

> In my data set, daggers have an ID of `2`, so I'm using that here as an example.
{: .prompt-info }

After refreshing the character detail view again, you should only see weapons of whichever type you filtered for in the dropdown for weapons in the HTML form. In my case, this means I only have one choice, the Bronze Dagger. However, our ultimate goal is to pull all weapons for weapon types that are allowed to be equipped for the character's current job. We want to use the `jobs_can_equip` field on the weapon type object, which we can do using the dunder (double underscore) syntax. In this case, Bilbo is a thief, which has an ID of 4.

```python
    weapon = serializers.PrimaryKeyRelatedField(
        queryset=Weapon.objects.filter(weapon_type__jobs_can_equip=4)
    )
```
{: file="characters/serializers.py" }

Refresh the character detail view again, and I once again only see the Bronze Dagger. This is because in my dataset, the only weapon a thief can equip is a dagger, and I only have one dagger in the system. If I change the job ID to something else, like 1 for the Warrior, the available items will change to reflect the change. However, we want this to filter for the current job of the character, not some specific job we hard code into it. So let's think about how we might do this. What we're looking for is something dynamic that we can pull from.

```python
    weapon = serializers.PrimaryKeyRelatedField(
        queryset=Weapon.objects.filter(weapon_type__jobs_can_equip=self.job)
    )
```
{: file="characters/serializers.py" }

Unfortunately, in this area where the fields are being defined, we don't have access to `self`, and even if we did, we would not necessarily have access to the bound object, even during initialization. There may be a way of hacking this directly in the `CharacterSerializer`, but doing so would likely be an anti-pattern and not the way Django Rest Framework expects things to be organized. Fortunately, this is where custom serializers can shine.

## Custom PrimaryKeyRelatedField Serializer

We want the functionality of the `PrimaryKeyRelatedField` but need to modify the way it behaves slightly to meet our needs. We can extend the class into our custom class to solve this problem. For consistency's sake, we'll want to create a custom class for the `Armor` objects as well, and since we already know how to modify the queryset for the `ArmorType` restriction, let's start with that.

I'm going to call my class the `DyanmicArmorSerializer` because I'm bad at naming things. We'll want the class to inherit from the `PrimaryKeyRelatedField` and we'll want to override the `get_queryset` function.

```python
class DynamicArmorSerializer(serializers.PrimaryKeyRelatedField):
    def __init__(self, **kwargs):
        super().__init__(**kwargs)

    def get_queryset(self):
        return super().get_queryset()
```
{: file="characters/serializers.py" }

When extending an existing class I like to outline what I need to do in this way, targeting the functions I'm going to need to override and using the `super()` functions to ensure the base class executes what it needs to. In this case, we're going to want to be able to pass in which armor slot the serializer is being used for, so we can pass that in through the keyword arguments in the `__init__` function. And we're going to want to override the queryset, so we'll go ahead and define that function and immediately pass since `PrimaryKeyRelatedField` does not have a default implementation of `get_queryset`.

Let's go ahead and update the `__init__` function to look for an `armor_slot` parameter.

```python
class DynamicArmorSerializer(serializers.PrimaryKeyRelatedField):
    def __init__(self, armor_slot=None, **kwargs):
        self.slot = armor_slot
        super().__init__(**kwargs)

    def get_queryset(self):
        pass
```
{: file="characters/serializers.py" }

Notice, that we accept `None` as a possible value. If an armor slot is provided, we can filter our queryset using that value. If it's none, we'll return all armor, without filtering it.

```python
class DynamicArmorSerializer(serializers.PrimaryKeyRelatedField):
    def __init__(self, armor_slot=None, **kwargs):
        self.slot = armor_slot
        super().__init__(**kwargs)

    def get_queryset(self):
        queryset = Armor.objects.all()

        if self.slot is not None:
            queryset = queryset.filter(armor_slot=self.slot)

        return queryset
```
{: file="characters/serializers.py" }

Let's go ahead and assign this serializer to our armor slots and see if it works. We'll need to pass in the slot type to each constructor, but we can use the `enum` we defined in the `Armor` class to pass these in easily.

```python
class CharacterSerializer(serializers.ModelSerializer):
    job = SimpleJobSerializer(many=False, read_only=True)
    stats = serializers.SerializerMethodField()

    weapon = serializers.PrimaryKeyRelatedField(
        queryset=Weapon.objects.filter(weapon_type__jobs_can_equip=self.job)
    )

    head = DynamicArmorSerializer(armor_slot=Armor.ArmorSlot.HEAD)
    body = DynamicArmorSerializer(armor_slot=Armor.ArmorSlot.BODY)
    hands = DynamicArmorSerializer(armor_slot=Armor.ArmorSlot.HANDS)
    feet = DynamicArmorSerializer(armor_slot=Armor.ArmorSlot.FEET)
```
{: file="characters/serializers.py" }

Another refresh of our character detail view, and we should see IDs for each piece of armor and should have validated dropdowns in the HTML form. Let's go ahead and create a `DynamicWeaponSerializer` so that we have the class ready for the next step. Since there is no specific slot to filter by, we don't need to override the initializer function and for now, we'll just return all weapons.

```python
class DynamicWeaponSerializer(serializers.PrimaryKeyRelatedField):
    def get_queryset(self):
        return Weapon.objects.all()

...

class CharacterSerializer(serializers.ModelSerializer):
    job = SimpleJobSerializer(many=False, read_only=True)
    stats = serializers.SerializerMethodField()

    weapon = DynamicWeaponSerializer()
    head = DynamicArmorSerializer(armor_slot=Armor.ArmorSlot.HEAD)
    body = DynamicArmorSerializer(armor_slot=Armor.ArmorSlot.BODY)
    hands = DynamicArmorSerializer(armor_slot=Armor.ArmorSlot.HANDS)
    feet = DynamicArmorSerializer(armor_slot=Armor.ArmorSlot.FEET)
```
{: file="characters/serializers.py" }

If we again refresh our character detail view, we should have a working weapon field that is updatable and more or less works exactly like the `PrimaryKeyRelatedField` does out of the box. Now let's tackle implementing our job filter.

### Let's Get a Job

Because we're basing our field on the `PrimaryKeyRelatedField` class, the parent class comes with a helpful property we can use to access the `Character` class these fields belong to. We can access the "parent" serializer - in this case the `CharacterSerializer` - by referring to `self.parent` within our custom field. This is set automatically by the `__init__` function of the `PrimaryKeyRelatedField` class, so we can simply access it within our `get_queryset` function override. The serializer has an `instance` field that refers to the object - in this case, a `Character` - that is being serialized. That `Character` object has the `job` field that we're looking for.

We'll start by modifying our `DyanmicWeaponSerializer` class since it will be the easiest to test. First, let's see where we're at by refreshing the character detail view and seeing what weapons are currently able to be equipped. In my dataset, I can see all the available weapons even though as a thief, Bilbo should only be able to equip the dagger. Now let's update our serializer and assign it.

```python
class DynamicWeaponSerializer(serializers.PrimaryKeyRelatedField):
    def get_queryset(self):
        character = self.parent.instance
        queryset = Weapon.objects.filter(
            weapon_type__jobs_can_equip=character.job
        )

        return queryset

...

class CharacterSerializer(serializers.ModelSerializer):
    job = SimpleJobSerializer(many=False, read_only=True)
    stats = serializers.SerializerMethodField()

    weapon = DynamicWeaponSerializer()
    head = DynamicArmorSerializer(armor_slot=Armor.ArmorSlot.HEAD)
    body = DynamicArmorSerializer(armor_slot=Armor.ArmorSlot.BODY)
    hands = DynamicArmorSerializer(armor_slot=Armor.ArmorSlot.HANDS)
    feet = DynamicArmorSerializer(armor_slot=Armor.ArmorSlot.FEET)

```
{: file="characters/serializers.py" }

If we now refresh the view, only the dagger is appearing as an option. Upon trying to assign the Bronze Sword to the weapon slot, an error is returned: `"Invalid pk \"1\" - object does not exist."`. Our validation is working! We can apply the same logic to our `DynamicArmorSerializer` and check those as well.

```python
class DynamicArmorSerializer(serializers.PrimaryKeyRelatedField):
    def __init__(self, armor_slot=None, **kwargs):
        self.slot = armor_slot
        super().__init__(**kwargs)

    def get_queryset(self):
        character = self.parent.instance
        queryset = Armor.objects.filter(
            armor_type__jobs_can_equip=character.job
        )

        if self.slot is not None:
            queryset = queryset.filter(armor_slot=self.slot)

        return queryset
```
{: file="characters/serializers.py" }

And there we have it! All our weapons and armors should now be restricted to the appropriate slot and jobs.

## Lost in the Details

While we have our validation working, we've lost all the simple details for the weapon and armor, which may be something we want to be able to still access. Because I want the retrieve and update schemas to match as closely as possible, I'm going to separate the field that is used to update the object assigned to the foreign key from its details. This will mean creating additional fields for details, which we'll name `{field}_details`.

However, this isn't as simple as just defining the fields and pointing them at the `SimpleArmorSerializer` and `SimpleWeaponSerializer`. Before when we were using these serializers, we defined them with the same field name as the model field. The serializers look for a field with a matching name when they're initialized and pull the data for serialization from that field. However, the name will no longer match now, so we'll have to specify which field needs to be serialized ourselves. The simplest way to do this is to use the `SerializerMethodField` and return the data from the serializer.

```python
class CharacterSerializer(serializers.ModelSerializer):
    job = SimpleJobSerializer(many=False, read_only=True)
    stats = serializers.SerializerMethodField()

    weapon = DynamicWeaponSerializer()
    weapon_details = serializers.SerializerMethodField()

    head = DynamicArmorSerializer(armor_slot=Armor.ArmorSlot.HEAD)
    head_details = serializers.SerializerMethodField()

    body = DynamicArmorSerializer(armor_slot=Armor.ArmorSlot.BODY)
    body_details = serializers.SerializerMethodField()

    hands = DynamicArmorSerializer(armor_slot=Armor.ArmorSlot.HANDS)
    hands_details = serializers.SerializerMethodField()

    feet = DynamicArmorSerializer(armor_slot=Armor.ArmorSlot.FEET)
    feet_details = serializers.SerializerMethodField()

    class Meta:
        model = Character
        fields = [
            'id',
            'job',
            'name',
            'description',
            'stats',
            'weapon',
            'weapon_details',
            'head',
            'head_details',
            'body',
            'body_details',
            'hands',
            'hands_details',
            'feet',
            'feet_details',
            'experience_points',
            'level'
        ]

    def get_stats(self, obj):
        ...
    
    def get_weapon_details(self, obj: Character) -> dict:
        return SimpleWeaponSerializer(obj.weapon, context=self.context).data

    def get_head_details(self, obj: Character) -> dict:
        return SimpleArmorSerializer(obj.head, context=self.context).data

    def get_body_details(self, obj: Character) -> dict:
        return SimpleArmorSerializer(obj.body, context=self.context).data

    def get_hands_details(self, obj: Character) -> dict:
        return SimpleArmorSerializer(obj.hands, context=self.context).data

    def get_feet_details(self, obj: Character) -> dict:
        return SimpleArmorSerializer(obj.feet, context=self.context).data
```
{: file="characters/serializers.py" }

One thing you might note is that we need to pass in the `context` field. This contains the `request` object which is needed to return absolute URLs instead of relative ones. Once this is added, your character detail view should look something like the following.

```json
{
    "id": 1,
    "job": {
        "id": 4,
        "name": "Thief",
        "detail_view": "http://127.0.0.1:8000/api/v1/characters/jobs/4/"
    },
    "name": "Bilbo",
    "description": "A very determined hobbit!",
    "stats": {
        "strength": 11,
        "dexterity": 12,
        "agility": 14,
        "vitality": 11,
        "intelligence": 11,
        "mind": 9,
        "attack": 14,
        "defense": 18,
        "magic": 10,
        "accuracy": 13,
        "evasion": 14,
        "speed": 21
    },
    "weapon": 2,
    "weapon_details": {
        "name": "Bronze Dagger",
        "detail_view": "http://127.0.0.1:8000/api/v1/characters/weapons/2/"
    },
    "head": 1,
    "head_details": {
        "name": "Leather Cap",
        "detail_view": "http://127.0.0.1:8000/api/v1/characters/armor/1/"
    },
    "body": 3,
    "body_details": {
        "name": "Leather Jerkin",
        "detail_view": "http://127.0.0.1:8000/api/v1/characters/armor/3/"
    },
    "hands": 2,
    "hands_details": {
        "name": "Leather Gloves",
        "detail_view": "http://127.0.0.1:8000/api/v1/characters/armor/2/"
    },
    "feet": 4,
    "feet_details": {
        "name": "Leather Boots",
        "detail_view": "http://127.0.0.1:8000/api/v1/characters/armor/4/"
    },
    "experience_points": 50,
    "level": 2
}
```
{: file="/api/v1/characters/1/" }

If you take a look at the HTML form, only the fields that use the `SerializedMethodField` do not show up in the HTML form, as they are read-only fields. They do show up in the template JSON data that Django provides, but their values will be ignored if you include them in a request. Only the writable fields are parsed when making a `PUT` or `PATCH` request. You'll notice that the `job` field currently isn't showing up in our form. This is because we haven't switched it from a read-only field yet. We can do that by adding a `job_details` field that points to the `SimpleJobSerializer` and removing the `job` field entirely. This will cause it to use the `PrimaryKeyRelatedField` by default. If you want to be explicit, you can directly assign it to that field type, but you will also need to include the `queryset` as a parameter.

```python
class CharacterSerializer(serializers.ModelSerializer):
    job_details = serializers.SerializerMethodField()
    stats = serializers.SerializerMethodField()

    ...

    class Meta:
        model = Character
        fields = [
            ...
            'job',
            'job_details',
            ...
        ]

    ...
    
    def get_job_details(self, obj: Character) -> dict:
        return SimpleJobSerializer(obj.job, context=self.context).data

    ...
```
{: file="characters/serializers.py" }

## Final Thoughts

And with that, we've added our field validation logic to our serializers. At this point, we have a fairly functional API for handling user data and are coming to the end of our series. Next week, we'll work on adding filters to our list views within our API, allowing us to filter which results we get back. This can be very useful in instances where you need to provide a curated set of options, like our weapons and armor. It can also be useful in instances where a search view might be built on top of the API endpoint, like with our character list view.
