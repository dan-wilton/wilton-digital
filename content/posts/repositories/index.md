---
title: 'repositories'
date: 2025-07-22
draft: true
description: 'an introduction to the repository pattern in Python'
tags:
  [
    'data',
    'technical',
    'work',
    'theory',
    'practical',
    'ddd',
    'domain',
    'design',
    'architecture',
    'repository',
    'workshop',
    'patterns',
  ]
series: ['a journey to domain-driven design']
series_order: 2
---

{{< alert >}}
**Background** <br/>
Bob works at a new up-and-coming tech company called "_Pear_" and has been hired to help automate some of Pear's manual processes. Pear have recently released their brand new flagship device, the PearBook Pro Max S+... and it's been a total 💩-show.

What's worse is that there was no expectation for anything to go wrong; the only way for customers to raise issues is to walk up to the Pear office, write down their issue on a post-it note, and slap it against the front door.

Bob is panicking. There are thousands of outraged nerds outside wanting their issues addressed, and there is only one Bob... Unforunately for us, _we are Bob_.
{{< /alert >}}

From the previous engineers that have worked at Pear, we've inherited a sophisticated system for creating Support Tickets:

1. Leave the office
2. Quickly grab one of the post-it notes (while trying to dodge tomatoes and verbal abuse)
3. Run back to your desk
4. Open the [Django Admin](https://www.w3schools.com/django/django_admin.php) panel
5. Manually add the information from the post-it note to the database

<br />

The existing Django `data` model currently looks like this:

```python
# src/data/models.py

from django.db import models


class Ticket(models.Model):
    description = models.CharField()
    author_email = models.EmailField(unique=True, db_index=True)
    last_updated_by = models.CharField()
```

<br />

While Django philosophy dictates the use of an ["Active Record" design pattern](https://docs.djangoproject.com/en/5.2/misc/design-philosophies/#include-all-relevant-domain-logic), the existing codebase splits out the domain logic from the data models.

If Bob wanted to expose an interface for other developers to retrieve a `Ticket` by the author's email. He might write a simple function like below:

```python
# src/domain/queries.py

from data import models as data_models
from django.db import models as django_models


def get_ticket_by_email(email: str) -> Ticket:
    return data_models.Ticket.objects.get(author_email=email)
```

<br />

But Bob is smarter than this... he knows it! He knows that this could create a number of issues:

- _Jim_ is the only other engineer in the company, and _makes Bob's life hell_. He'll use this function to retrieve a `Ticket`, change it in many stupid ways, and call `.save()` on it - Bob would be none-the-wiser.
- _Jim_ always updates the `description`, but never updates the `last_updated_by` field. With the current function and exposing the data model, Bob can't enforce a rule to stop this.
- With rumours of a new data source being introduced at Pear, Bob knows that tying all of his functions to the Django models _could_ mean a huge refactor in the future.

As Bob, we need a solution that allows us to give a non-django object back to Jim - which is quite easy. But we also need a solution that adds a layer of abstraction between our database operations and our business logic. This would allow us to swap out our data source if we ever needed to, while keeping our business logic that stops Jim creating all sorts of trouble.

This is where ✨ **Repositories** ✨ come in. First, we can start by defining a non-django-based `Ticket` entity - we don't want any knowledge of the database interactions, or any of the methods that the `data` version provides:

```python
# src/domain/entities.py

class Ticket:
    _description: str
    _author_email: str
    _last_updated_by: str

    def __init__(
        self,
        description: str,
        author_email: str,
        last_updated_by: str,
    ) -> None:
        self._description = description
        self._author_email = author_email
        self._last_updated_by = last_updated_by

    @property
    def description(self) -> str:
        return self._description

    @property
    def author_email(self) -> str:
        return self._author_email

    @property
    def last_updated_by(self) -> str:
        return self._last_updated_by
```

<br />

As our data source could change at any moment, we want to define the interface for retrieving and saving our entities in our `domain` layer - that way we can be sure that any implementation of the `repository` we retrieve our `Ticket` entity from will always be shaped the same.

We can do this by creating an abstract repository in the `domain` layer, bearing in mind we don't need to know how it is actually implemented or how data is accessed:

```python
# src/domain/repositories.py
import abc

from . import entities


class TicketRepository(abc.ABC):
    @abc.abstractmethod
    def get(self, email: str) -> entities.Ticket: ...

    @abc.abstractmethod
    def save(self, ticket: Ticket) -> entities.Ticket: ...

    @abc.abstractmethod
    def delete(self, ticket: Ticket) -> None: ...
```

<br />

In our `data` layer, where we can care about our django db interactions, we can start to implement a concrete version of this abstract repository:

```python
# src/data/repositories.py
from domain import repositories as domain_repositories
from domain import entities as domain_entities

from . import models


class DjangoTicketRepository(domain_repositories.TicketRepository):
    def get(self, author_email: str) -> entities.Ticket:
        ticket_data_model = models.Ticket.objects.get(
            author_email=author_email,
        )

        # Map data model to our domain entity
        return domain_entities.Ticket(
            description=ticket_data_model.description,
            author_email=ticket_data_model.author_email,
        )

    def save(self, ticket: Ticket) -> entities.Ticket:
        ticket_data_model = models.Ticket(
            description=ticket.description,
            author_email=ticket.author_email,
        )
        ticket_data_model.save()
        return ticket

    def delete(self, author_email: str) -> None:
        ticket_data_model = models.Ticket.objects.get(
            author_email=author_email,
        )
        ticket_data_model.delete()
        return
```

<br />
At this point, you might be wondering "why the hell do I need to write so much code for something that was only a few lines previously?" and you're not wrong. As with all patterns, there are pros, cons and balances to be sought-after. This is a very simple example, but as the complexity grows, we should start to see some of the benefit. 👀

The first benefit being that Bob can now start to enforce the business' rules on poor unsuspecting Jim...

```python
# src/domain/entities.py

class Ticket:
    ...

    def update_description(
        self,
        new_description: str,
        updated_by: str,
    ) -> None:
        self._last_updated_by = updated_by
        self._description = new_description
```

Using linting/importing rules, Bob can ban Jim from reading the `data` models directly. Any time he'd want to update a description, he'd have to do something like the following:

```python
# jims_update_description_script.py

from data import repositories


if __name__ == "__main__":
    repository = repositories.DjangoTicketRepository()

    author_email_of_ticket_to_update = "test@wilton.digital"

    ticket = repository.get(author_email_of_ticket_to_update)
    ticket.update_description(
        new_description="Modified description"
        updated_by="definitely _not_ Jim",
    )

    repository.save(ticket)
```

Bob can rest easy knowing that the business rules will always be followed, _and_ he can even browse his code a bit easier! Code belonging to data access and database operations are in the `data` module, code that reflects the real-life business rules and domain concepts are in the `domain` module.
