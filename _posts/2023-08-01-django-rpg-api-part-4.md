---
title: Creating an RPG API in Python Django (Part 4) - Jobs, Stats, and Foreign Keys
description: We start building out our stat and job systems and explore implementing foreign keys within our data.
date: 2023-08-01 06:00:00 -0400
categories: [Development, Python]
tags: [python, django, database, rest api, postgresql]
img_path: /assets/images/posts/django-rpg-api-part-4/
image:
  path: hero.jpg
  alt:
math: true
---

In our [last post]({% post_url 2023-07-11-django-rpg-api-part-3 %}), we setup our experience points system, allowing our characters to level based on the number of experience points they've gained. This week, we'll begin creating the various stats that will effect their abilities in combat and tie those stats to specific jobs. This means we will need to define a job model and link it back to each character via a foreign key, and then learn how to represent that relationship within our serializers and display it within our existing character endpoint.

> If you'd like to follow along, you can find the [complete source code from last week on GitHub](https://github.com/jmbarne3/rpgapi/tree/v0.3.0). Clone down the repository and follow the setup instructions in the readme.
{: .prompt-info }

We're going to be moving a little more quickly this week than we have in past weeks. Now that we have the basics of model and serializer creation down, the initial steps to getting our jobs setup should be fairly quick. We'll take our time as we explore connecting the two models, but won't dwell on the creation of the new model itself. Because we'll be moving a little faster, I think it will be helpful to list out our goals for this week:

1. Create a Job model that defines stat growth for each character depending on which Job they have
2. Connect the Job model to our Character model and explore different ways to represent the relationship through our serializer
3. Create properties or fields (we'll explore the decision of which to use again) that represent our stats

## Defining Our Stats

Before we can build our model, we should define what we're building clearly. I want to build a system where each of 6 basic stats are influenced by the current Job of a character. In addition, each character will have 6 additional stats that are effected by the 6 base stats and are additionally affected by equipped items, which we'll build out in a future post.

These stats could be anything and could represent anything you want. I will be using the following 6 stats which are based on 6 of the 7 stats from Final Fantasy XI.

| Stat Name | Effect |
| --- | --- |
| Strength | Increases physical attack of both melee and ranged attacks |
| Dexterity | Increases melee accuracy and critical hit rate |
| Vitality | Increases defense and effects total hit points |
| Agility | Increases evasion, ranged accuracy and character's speed |
| Intelligence | Increases magic accuracy, black magic potency and effects total magic points |
| Mind | Increases magic accuracy, white magic potency and effects total magic points |

In addition to these 6 base stats, we'll be using 6 stats that directly play into our battle calculations. These stats are effected by the base stats above and are modified by equipped items. For example, weapons will add to our attack stat and pieces of armor will add to our defense stat. Lighter armor might add evasion and speed, where heavy armor might reduce those stats. Magical weapons and armor might effect the magic stat. If we were building out a complete battle system, we might focus on these calculated stats when it comes to status effects, both beneficial and enfeebling. 

| Stat Name | Effect |
| --- | --- |
| Attack | Determines the amount of physical damage dealt by melee and ranged weapons |
| Defense | Determines the amount of physical damage taken by melee and ranged weapons |
| Magic | Determines the amount of magical damage dealt and taken by spells |
| Accuracy | Determines the likely hood of landing a melee attack or ability |
| Evasion | Determines the likely hood of a melee attack missing |
| Speed | Determines the order in which characters take their turns |

This should give us a nice base of stats to work with as an example. While I've not entirely thought through how the system would work in reality, this should give us plenty of levers to pull when designing our Jobs to allow for some variety between them. Since the purpose of this project isn't actually to design a working RPG combat system, the specific details aren't really important. However, it's always more fun to work on projects that have a little practical thought behind their design so that you force yourself to work through real-world problems that might arise when working on them.

## Designing Our Model

Our Job model will be fairly simple to begin with. We'll want each job to have a name, a description and then some way of effecting the the values of our six base stats. For simplicity sake, we'll be using a linear function for stat growth instead of an exponential function like we did with experience points. However, whatever formula you use, you will likely want a certain level of precision, so we'll use a `FloatField` to store a constant that will effect how quickly or slowly a particular stat grows for that job. With those basic parameters worked out, you can add the following model to you `characters/models.py` file.

```python
class Job(models.Model):
    name = models.CharField(max_length=255, null=False, blank=False)
    description = models.TextField(null=True, blank=True)
    strength_mod = models.FloatField(null=False, blank=False)
    dexterity_mod = models.FloatField(null=False, blank=False)
    vitality_mod = models.FloatField(null=False, blank=False)
    agility_mod = models.FloatField(null=False, blank=False)
    intelligence_mod = models.FloatField(null=False, blank=False)
    mind_mod = models.FloatField(null=False, blank=False)

    def __str__(self):
        return self.name
```
{: file='characters/models.py' }

This will provide the basic structure of our model and is a great starting point. Since the `Character` class will be referencing this class in a few moments, you will want to make sure you create the `Job` class _before_ the `Character` class in your code. Let's go ahead and add a field to our `Character` class that identifies which job the character has. Since a character can only have one job at a time, but multiple characters could have the same job, we'll define this as a many-to-one relationship. To do this, we need the ForeignKey to be on the `Character` model/table.

```python
class Character(models.Model):
    name = models.CharField(max_length=255, null=False, blank=False)
    description = models.TextField(null=True, blank=True)
    experience_points = models.IntegerField(null=False, blank=False, default=0)
    level = models.IntegerField(null=False, blank=False, default=1, editable=False)
    job = models.ForeignKey(Job, on_delete=models.CASCADE)
```
{: file="characters/models.py" }

### Foreign Key Constraints

Let's go ahead and try to create our migration for these changes and apply them using the command `python manage.py makemigrations && python manage.py migrate`. A Foreign Key cannot be null by default, so just like some of our other fields we've added, we'll need to provide a default value as part of our migration. However, we don't currently have any jobs that we've created that we can use as a default. This poses an interesting - and frustratingly common - dilemma that we need to work through in order to get this new field into our database.

There are a number of ways to tackle adding a new field like this. Some developers opt to make the foreign key nullable for the initial migration and then make it not nullable in a future migration. This is a valid approach, however, it makes certain assumptions about the data within the application that are not well defined within the application itself. Say, for example, you create a new foreign key field that links to the class `Widget` and set it to be nullable. You then get that code into production and your users create a few new `Widget` objects, one of which works nicely as a standard default and has a primary key of 3. So in your next sprint, you make your field not nullable, and set the default - either in the model or in the migration - to the value 3, which in your production system points to the `Widget` you want to use as your default.

Some of you may already see the problem with this. If someone else pulls down your code and runs the migrations against their brand new database, there will be no `Widget` with an ID of 3. The migration will fail, since the foreign key constraint can't be enforced if the record doesn't exist. One interesting way to approach the problem - and the one we'll be using here - is to use the migration itself to ensure a single default object always exists and always has a particular ID.

### Custom Migrations

To implement this work around, we're going to design a custom migration. There are several of ways to do this, and we will be going over the two I find most useful. The first is to create a completely blank migration where you can then provide all the migration logic yourself. To do this, you would use the following command: `python manage.py makemigrations <app_name> --empty`. When creating an empty migration, it's important to provide the app name if you have multiple apps in your project. Django can't infer where to create the migration file if you don't tell it which app it's for. Once you create the migration, you'll have a new file in the migrations directory for your application which will contain the minimum required syntax for a migration.

```python
from django.db import migrations


class Migration(migrations.Migration):

    dependencies = [
        ('characters', '0002_some_previous_migration'),
    ]

    operations = [
    ]
```
{: file='characters/migrations/0003_empty_migration.py' }

> If you tried out this command on your local instance, just be sure to delete the migration file before moving onto any of the further steps so that you're not left with an empty migration in your project.
{: .prompt-warning }

Once you have this file you can begin adding operations to the `operations` array for each change you would like made. Going over the specifics of migration operations is a little out of the scope of what we're doing here, but know that if you need to do something completely from scratch, this is a good starting place.

To achieve our goal of creating a default value within our migration, we're going to let Django generate the schema modification portions of the migrations and then add our own custom functions for creating and deleting a default value.

> Since migrations can be rolled back, it's important to always create a reverse operation for a custom action if that action needs to be rolled back with the schema.
{: .prompt-info }

First, have Django generate a new migration based on the model changes, but do not apply the migration yet: `python manage.py makemigrations characters`. This should create a new migration file that looks something like this:

```python
from django.db import migrations, models
import django.db.models.deletion

class Migration(migrations.Migration):
    dependencies = [
        ('characters', '0002_character_experience_points_character_level'),
    ]

    operations = [
        migrations.CreateModel(
            name='Job',
            fields=[
                ('id', models.BigAutoField(auto_created=True, primary_key=True, serialize=False, verbose_name='ID')),
                ('name', models.CharField(max_length=255)),
                ('description', models.TextField(blank=True, null=True)),
                ('strength_mod', models.FloatField()),
                ('dexterity_mod', models.FloatField()),
                ('vitality_mod', models.FloatField()),
                ('agility_mod', models.FloatField()),
                ('intelligence_mod', models.FloatField()),
                ('mind_mod', models.FloatField()),
            ],
        ),
        migrations.AddField(
            model_name='character',
            name='job',
            field=models.ForeignKey(default=1, on_delete=django.db.models.deletion.CASCADE, to='characters.job'),
            preserve_default=False,
        ),
    ]

```
{: file="characters/migrations/0003_job_character_job.py" }

If you've been looking over the other migrations we've generated, this should look pretty standard. The foreign key field type has [a few extra bits that it would be good to read up on](https://docs.djangoproject.com/en/4.2/ref/models/fields/#django.db.models.ForeignKey), but is other-wise pretty standard. To this migration, we want to add two functions. The first function will add a new `Job` object with an ID of 1 to our database and the second function will remove it.

```python
from django.db import migrations, models
import django.db.models.deletion

def create_default_job(apps, schema_editor):
    Job = apps.get_model('characters', 'Job')
    Job(
        id=1,
        name='Freelancer',
        description='Default job for all new characters',
        strength_mod=1.0,
        dexterity_mod=1.0,
        vitality_mod=1.0,
        agility_mod=1.0,
        intelligence_mod=1.0,
        mind_mod=1.0
    ).save()

def remove_default_job(apps, schema_editor):
    Job = apps.get_model('characters', 'Job')
    try:
        Job.objects.get(pk=1).delete()
    except Job.DoesNotExist:
        print("Default job did not exist so was not deleted.")

class Migration(migrations.Migration):
    ...
```
{: file="characters/migrations/0003_job_character_job.py" }

Because certain classes may or may not exist when a migration is run, it's important to refer to models through the `apps` object that's passed to migration functions. More can be read about both the `apps` and `schema_editor` objects in the [documentation for the `RunPython`](https://docs.djangoproject.com/en/4.2/ref/migration-operations/#runpython) class which will be calling these functions. Once we have a reference to the `Job` model we can create the default object using the normal initialization method we would use elsewhere and save it. To remove the object, we go ahead and use a chained `get().delete()` wrapped in a `try/except` just to make sure it doesn't exception out if the object doesn't exist.

To hook these operations into the migration, we'll want to use the `RunPython` function provided by the `migrations` module. We'll insert a call to `RunPython` right in between our new model creation and the modification of the `Character` object to include the new `ForeignKey`.

```python
from django.db import migrations, models
import django.db.models.deletion

def create_default_job(apps, schema_editor):
    Job = apps.get_model('characters', 'Job')
    Job(
        id=1,
        name='Freelancer',
        description='Default job for all new characters',
        strength_mod=1.0,
        dexterity_mod=1.0,
        vitality_mod=1.0,
        agility_mod=1.0,
        intelligence_mod=1.0,
        mind_mod=1.0
    ).save()

def remove_default_job(apps, schema_editor):
    Job = apps.get_model('characters', 'Job')
    try:
        Job.objects.get(pk=1).delete()
    except Job.DoesNotExist:
        print("Default job did not exist so was not deleted.")


class Migration(migrations.Migration):

    dependencies = [
        ('characters', '0002_character_experience_points_character_level'),
    ]

    operations = [
        migrations.CreateModel(
            name='Job',
            fields=[
                ('id', models.BigAutoField(auto_created=True, primary_key=True, serialize=False, verbose_name='ID')),
                ('name', models.CharField(max_length=255)),
                ('description', models.TextField(blank=True, null=True)),
                ('strength_mod', models.FloatField()),
                ('dexterity_mod', models.FloatField()),
                ('vitality_mod', models.FloatField()),
                ('agility_mod', models.FloatField()),
                ('intelligence_mod', models.FloatField()),
                ('mind_mod', models.FloatField()),
            ],
        ),
        migrations.RunPython(
            create_default_job,
            remove_default_job
        ),
        migrations.AddField(
            model_name='character',
            name='job',
            field=models.ForeignKey(default=1, on_delete=django.db.models.deletion.CASCADE, to='characters.job'),
            preserve_default=False,
        ),
    ]

```
{: file="characters/migrations/0003_job_character_job.py" }

And that's our complete migration code. If you run the migration, the `Job` table will get created, followed the creation of the default job and then an alter statement will be run against the `Character` table adding the foreign key. If you rollback the migration - `python manage.py migrate characters 0002` - the reverse of each operation will run in reverse order. So the foreign key and column will be removed, our `remove_default_job` function will then run followed by the deletion of the `Job` table.

## Choices

We could leave our `Job` model like this and it would be perfectly functional. However, there's still the detail of what our stat growth formula is going to be and how we're going to control different levels of growth. One common model that is used in RPGs is to standardize an array of growth levels that in turn correspond to some constant that's used in the growth formula. So for example, you might see something that looks like the following:

| Growth Rate | Starting Amount (Lvl 1) | Ending Amount (Lvl 99) |
| --- | --- | --- |
| A | 10 | 100 |
| B | 9 | 90 |
| C | 7 | 75 |
| D | 5 | 60 |
| E | 4 | 45 |
| F | 2 | 30 |

This example is probably not very practical, but hopefully it conveys the idea. Instead of having to come up with specific constants for growth for each stat for each job, we can instead create an array of constants with a nice human readable name (in this case a "grade" of sorts) and then assign each job a "growth rate" for each stat. Warriors might get an A in strength and vitality, but an F in intelligence. Mages on the other hand might have As and Bs for intelligence and mind, but lower rates of growth for physical stats.

In Django, we can easily map a set of human-readable choices to a desired data type using the `choices` parameter available in most field types. You need to define an array of tuples, which each tuple containing the stored value and then the display value of each choice, and then pass that array to the `choices` parameter. Alternatively, you can create an enumeration and pass it to the choices field. In this case, we're going to use a custom Django type that derives from the `models.Choices` class that will allow us to use the values like their an enumeration but correctly translates them to the appropriate type when added into the choices field.

```python
class Job(models.Model):

    class GrowthRate(float, models.Choices):
        A = 2.0, 'A'
        B = 1.8, 'B'
        C = 1.6, 'C'
        D = 1.4, 'D'
        E = 1.2, 'E'
        F = 1.0, 'F'

    name = models.CharField(max_length=255, null=False, blank=False)
    description = models.TextField(null=True, blank=True)
    strength_mod = models.FloatField(null=False, blank=False, choices=GrowthRate.choices)
    dexterity_mod = models.FloatField(null=False, blank=False, choices=GrowthRate.choices)
    vitality_mod = models.FloatField(null=False, blank=False, choices=GrowthRate.choices)
    agility_mod = models.FloatField(null=False, blank=False, choices=GrowthRate.choices)
    intelligence_mod = models.FloatField(null=False, blank=False, choices=GrowthRate.choices)
    mind_mod = models.FloatField(null=False, blank=False, choices=GrowthRate.choices)

    def __str__(self):
        return self.name
```
{: file="characters/models.py" }

Because Django does not offer a `FloatChoices` field, we need to construct one by subclassing our `GrowthRate` class to both `float` and `models.Choices`. We are then able to pass the property `.choices` to the `choices` parameter on each `FloatField` in our Job model. This change does require creating a new migration, although nothing on the database will actually change. However, not creating the migration will cause Django to constantly throw a warning as you run things, so I generally go ahead and create the migration just to quiet the warning.

## Serializers

We now have our new model and we have our foreign key pointing to our new model. Let's open up one of our API endpoints and see what everything looks like!

```json

[
    {
        "id": 1,
        "name": "Bilbo",
        "description": "A very decent hobbit, indeed!",
        "experience_points": 50,
        "level": 2,
        "job": 1
    }
]
```
{: file="/api/v1/characters/" }

Okay, well... That was a little underwhelming. Let's do a few things to give us some options here. We can, first, create a serializer for our `Job` model and create some endpoints for it. 

> Since we've covered most of these steps in past posts, I will not be explaining each new block of code going forward unless we're doing something new in one of them.
{: .prompt-info }

```python
from .models import Character, Job

class JobSerializer(serializers.ModelSerializer):
    class Meta:
        model = Job
        fields = '__all__'
```
{: file="characters/serializers.py" }
_Add the Job class to your imports and create your new serializer._

```python
from .models import Character, Job
from .serializers import CharacterSerializer, JobSerializer

# Create your views here.
class JobListView(generics.ListAPIView):
    queryset = Job.objects.all()
    serializer_class = JobSerializer

class JobDetailView(generics.RetrieveAPIView):
    queryset = Job.objects.all()
    lookup_field = 'id'
    serializer_class = JobSerializer
```
{: file="characters/views.py" }
_Add the Job and JobSerializer classes to your imports and create two new read-only views._

```python
from django.urls import path

from .views import (
    CharacterListView,
    CharacterDetailView,
    JobListView,
    JobDetailView
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
_Import your new views, add and rearrange the paths to provide for all the new endpoints. Because we're catching a lot of URLs greedily with `<id>/`, we need to put it below the two URLs for jobs._

Once all of this is added, you should be able to browse to `/api/v1/character/jobs/` and specifically `/api/v1/character/jobs/1/` and see at least the default Freelancer job. I encourage you to go ahead and add a few additional jobs to your dataset so you have some data to play around with.

{% details If you're interested in which Jobs I created, you can click here. %}
```json
[
    {
        "id": 1,
        "name": "Freelancer",
        "description": "Default job for all new characters",
        "strength_mod": 1.0,
        "dexterity_mod": 1.0,
        "vitality_mod": 1.0,
        "agility_mod": 1.0,
        "intelligence_mod": 1.0,
        "mind_mod": 1.0
    },
    {
        "id": 2,
        "name": "Warrior",
        "description": "Hits things",
        "strength_mod": 2.0,
        "dexterity_mod": 1.8,
        "vitality_mod": 1.8,
        "agility_mod": 1.6,
        "intelligence_mod": 1.4,
        "mind_mod": 1.4
    },
    {
        "id": 3,
        "name": "Monk",
        "description": "Hits things with fists",
        "strength_mod": 1.8,
        "dexterity_mod": 1.8,
        "vitality_mod": 2.0,
        "agility_mod": 1.8,
        "intelligence_mod": 1.2,
        "mind_mod": 1.6
    },
    {
        "id": 4,
        "name": "Thief",
        "description": "Steals things",
        "strength_mod": 1.6,
        "dexterity_mod": 1.8,
        "vitality_mod": 1.6,
        "agility_mod": 2.0,
        "intelligence_mod": 1.6,
        "mind_mod": 1.4
    },
    {
        "id": 5,
        "name": "Red Mage",
        "description": "Meh",
        "strength_mod": 1.6,
        "dexterity_mod": 1.6,
        "vitality_mod": 1.6,
        "agility_mod": 1.6,
        "intelligence_mod": 1.6,
        "mind_mod": 1.6
    },
    {
        "id": 6,
        "name": "Black Mage",
        "description": "Glass cannon",
        "strength_mod": 1.4,
        "dexterity_mod": 1.4,
        "vitality_mod": 1.4,
        "agility_mod": 1.6,
        "intelligence_mod": 2.0,
        "mind_mod": 1.8
    },
    {
        "id": 7,
        "name": "White Mage",
        "description": "Heals all the things",
        "strength_mod": 1.4,
        "dexterity_mod": 1.4,
        "vitality_mod": 1.6,
        "agility_mod": 1.4,
        "intelligence_mod": 1.8,
        "mind_mod": 2.0
    }
]
```
{: file="/api/v1/characters/jobs/" }
{% enddetails %}

### Updating Character Serializer

Now let's go back to our character serializer and explore our options. We could leave things just the way they are. Developers using our API would need to get the ID of the job from our `job` field and pass it to the `/api/v1/characters/jobs/<id>` endpoint to get the details. But we have some other options we can provide.

First, we _could_ provide a complete copy of each job for each character. To do this, we'd simply reuse the serializer we've just created for the `Job` model and add it to our `Character` serializer like so:

```python
class CharacterSerializer(serializers.ModelSerializer):
    job = JobSerializer(many=False)

    class Meta:
        model = Character
        fields = '__all__'
```
{: file="characters/serializers.py" }

Now when we refresh one of our endpoints using the Character serializer, we should see something that looks like this:

```json
{
    "id": 1,
    "job": {
        "id": 1,
        "name": "Freelancer",
        "description": "Default job for all new characters",
        "strength_mod": 1.0,
        "dexterity_mod": 1.0,
        "vitality_mod": 1.0,
        "agility_mod": 1.0,
        "intelligence_mod": 1.0,
        "mind_mod": 1.0
    },
    "name": "Bilbo",
    "description": "A very determined hobbit!",
    "experience_points": 50,
    "level": 2
}
```
{: file="/api/v1/characters/1/" }

We've been able to add quite a bit of detail here. If we were building a system where there were a lot of "jobs" and a lot of variety - meaning usually only a few characters might have the same job - then it might make a lot of sense to display the data like this. Developers on the frontend are very likely going to have to retrieve the job data anyway to make sure of the character data, and the data doesn't repeat very often, so we're not unnecessarily bloating our responses with redundant data.

In this case, however, we likely have a very small number of jobs and a lot of overlap between players having the same jobs. Instead of using our `JobSerializer`, we can use a simpler serializer that will provide less data, while still giving us what we need to display basic information without having to make another API call.

There are a [variety of options](https://www.django-rest-framework.org/api-guide/relations/) that Django Rest Framework provides for representing related fields (i.e. ForeignKey fields). We'll explore two of the most commonly used related field serializers and then will take a look at creating a simple related field of our own.

### String Related Field

The first field we can try out is the [String Related Field](https://www.django-rest-framework.org/api-guide/relations/#stringrelatedfield). This will give us a string representation of the object (in this case the job name), but no other information. To implement it, we can swap out the `JobSerializer` with the `StringRelatedField`.

```python
class CharacterSerializer(serializers.ModelSerializer):
    job = serializers.StringRelatedField(many=False)

    class Meta:
        model = Character
        fields = '__all__'
```
{: file="characters/serializers.py" }

This results in a nice simple representation of the job in our character detail view:

```json
{
    "id": 1,
    "job": "Freelancer",
    "name": "Bilbo",
    "description": "A very determined hobbit!",
    "experience_points": 50,
    "level": 2
}
```
{: file="/api/v1/characters/1/" }

However, we currently have no way of retrieving additional information about the job using the job's name. We could change this by adding a view that takes in the name as a parameter and returns back the job's details, but as a job could - and in my example data _does_ - have things like spaces or other characters that are not standard to URLs, this is usually not the way APIs are designed.

### Hyperlinked Related Field

Alternatively, we could link out to our JobDetail view using a [HyperlinkedRelatedField](https://www.django-rest-framework.org/api-guide/relations/#hyperlinkedrelatedfield). In this case, none of the information related to the job would be accessible directly in the character view, but we'd have the exact API endpoint we need to reach in order to get the information.

```python
class CharacterSerializer(serializers.ModelSerializer):
    job = serializers.HyperlinkedRelatedField(
        view_name='api.characters.jobs.detail',
        lookup_field='id',
        read_only=True,
        many=False
    )

    class Meta:
        model = Character
        fields = '__all__'
```
{: file="characters/serializers.py" }

This results in our Character endpoint looking something like this:

```json
{
    "id": 1,
    "job": "http://127.0.0.1:8000/api/v1/characters/jobs/1/",
    "description": "A very determined hobbit!",
    "experience_points": 50,
    "level": 2
}
```
{: file="/api/v1/characters/1/" }

This gives us all the information we need to retrieve the job information, but doesn't give us immediate access to anything in particular that we might want to display. For example, we might not need all the job data when displaying a list of characters, but we might want to at least know what the _name_ of the character's current job is. We can solve this by creating a simpler custom serializer and assigning it to our Character serializer.

### Custom Serializer

This will work just like the JobSerializer we created earlier, just with fewer fields defined. We want to provide as much information as is needed and link out to everything else. In this case, we'll create a serializer that provides the ID, the name and a URL to the detail view.

```python
from rest_framework import serializers
from rest_framework.reverse import reverse

from .models import Character, Job

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

class CharacterSerializer(serializers.ModelSerializer):
    job = SimpleJobSerializer(many=False)

    class Meta:
        model = Character
        fields = '__all__'
```
{: file="characters/serializers.py" }

In our simple serializer (which looks a lot more complicated than it really is), we configure the class to use the Job model as our model and define three fields: `id`, `name` and `detail_view`. The first two fields are part of our model, so there's nothing to do for those. However, the `detail_view` is not a defined field, so we need to define it. We do so on the very first line after the class declaration, defining it as a [`SerializerMethodField`](https://www.django-rest-framework.org/api-guide/fields/#serializermethodfield). This is a special kind of field in Django Rest Framework that allows us to define through a function what data should be returned for that field. The function has to be named `get_{field_name}` and is read-only. In the code above, we define the `get_detail_view` function and pass in the `obj` as a parameter. This is the current `Job` being represented. Since we want the URL of the specific job, we can use the `reverse` function to generate the URL and return it back.

> Note that I am using the Django Rest Framework version of `reverse` in the example above and _not_ the function found in the `django.urls` module. When you pass the current request to the function - which is found in the `rest_framework.reverse` module - it will prepend on the schema and domain to provide you with an absolute URL instead of a relative.
{: .prompt-info }

If we refresh our Character detail view now, we should see something like the following:

```json
{
    "id": 1,
    "job": {
        "id": 1,
        "name": "Freelancer",
        "detail_view": "http://127.0.0.1:8000/api/v1/characters/jobs/1/"
    },
    "name": "Bilbo",
    "description": "A very determined hobbit!",
    "experience_points": 50,
    "level": 2
}
```
{: file="/api/v1/characters/1/" }

This is a nice compromise between having too much data and not enough. Because we've provided the ID, name and detail view URL, we now have options. For example, on a search or simple list, we can just display the name and not use the additional data tied to the job. On a view where we might want additional job information, we can retrieve it _once_ and cache it using the ID of the job, avoiding having to retrieve it again when it shows up in the data after the first time.

## Final Thoughts

At this point, we've pretty well explored the basics of working with foreign keys within a Django project. There are more specifics when it comes to working with `ManyToMany` fields or when foreign keys are writable within views and serializers, but in most cases where the detail view is writable and it's related representations are not (e.g. while a character's _job_ can be changed, details about the job cannot be changed from the character views), most of the work revolves around finding ways to represent the data in ways that are efficient while still being useful.

In our next post, we'll incorporate our new Job data into the Character class and build out our stats. We'll then finish off the series by creating equippable items - which will give us a chance to play with data validation - and then look at writing some tests for the project. See you next week!
