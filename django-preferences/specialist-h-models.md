## SPECIALIST H: Models

### Abstract Base Models

```python
# common/models.py
import uuid
from django.db import models
from django.utils import timezone


class UUIDModel(models.Model):
    """UUID primary key for external-facing resources."""
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)

    class Meta:
        abstract = True


class TimestampedModel(models.Model):
    created_at = models.DateTimeField(auto_now_add=True, db_index=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True
        ordering = ["-created_at"]


class SoftDeleteQuerySet(models.QuerySet):
    def delete(self):
        return self.update(deleted_at=timezone.now())

    def hard_delete(self):
        return super().delete()

    def alive(self):
        return self.filter(deleted_at__isnull=True)

    def dead(self):
        return self.filter(deleted_at__isnull=False)


class SoftDeleteManager(models.Manager):
    def get_queryset(self):
        return SoftDeleteQuerySet(self.model, using=self._db).alive()


class SoftDeleteModel(models.Model):
    deleted_at = models.DateTimeField(null=True, blank=True, default=None)

    objects = SoftDeleteManager()     # alive() by default
    all_objects = models.Manager()    # includes deleted

    def delete(self, using=None, keep_parents=False):
        self.deleted_at = timezone.now()
        self.save(update_fields=["deleted_at"])

    def hard_delete(self):
        super().delete()

    class Meta:
        abstract = True
```

### Model Naming: `<Domain>DB`

Django ORM models are suffixed with `DB` to distinguish them from domain models (plain Pydantic) and to make the persistence layer explicit at a glance.

```python
# apps/orders/models.py

class OrderDB(UUIDModel, TimestampedModel, SoftDeleteModel):
    """Persistence model. Never import this outside the orders module.
    Cross-module access goes through OrderClient."""

    # No ForeignKey to UserDB — store the ID only (see cross-module rule below)
    customer_id = models.IntegerField(db_index=True)

    status = models.CharField(
        max_length=20,
        choices=OrderStatus,
        default=OrderStatus.DRAFT,
        db_index=True,
    )
    total = models.DecimalField(max_digits=10, decimal_places=2)

    class Meta(TimestampedModel.Meta):
        db_table = "orders_order"
        indexes = [
            models.Index(fields=["customer_id", "status"]),
            models.Index(fields=["status", "created_at"]),
        ]
        constraints = [
            models.CheckConstraint(
                check=models.Q(total__gte=0),
                name="order_total_non_negative",
            )
        ]

    def __str__(self) -> str:
        return f"OrderDB {self.pk} ({self.status})"
```

**Cross-module FK rule:** Never define a `ForeignKey` from one module's `DB` model to another module's `DB` model. Store the ID as a plain `IntegerField` (or `UUIDField`). Referential integrity at the application layer is enforced through the `Client` interface. This keeps modules independently deployable and avoids cross-module Django migration dependencies.

### TextChoices

```python
class OrderStatus(models.TextChoices):
    DRAFT = "draft", "Draft"
    PENDING = "pending", "Pending"
    CONFIRMED = "confirmed", "Confirmed"
    SHIPPED = "shipped", "Shipped"
    CANCELLED = "cancelled", "Cancelled"
```

**Rules:**
- Always use `TextChoices` (not raw string constants). IDE-safe, self-documenting.
- Add `db_index=True` to any `choices` field used in production `filter()` calls.
- Third string is the display label — always provide it.

### Custom Managers — Keep `objects`

```python
class PublishedManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(is_published=True)

class Article(TimestampedModel):
    is_published = models.BooleanField(default=False)

    objects = models.Manager()       # always keep — breaks admin if removed
    published = PublishedManager()   # scoped manager as named attribute
```

**Rule:** Never replace `objects` with a custom manager as the sole manager. Keep `objects = models.Manager()` and add scoped managers as additional named attributes.

### Meta Conventions

```python
class Meta:
    db_table = "app_modelname"       # explicit — no magic generation
    ordering = ["-created_at"]       # always set; DB default is fragile
    verbose_name = "order"
    verbose_name_plural = "orders"
    indexes = [...]
    constraints = [...]
```

### Migration Discipline

- Commit the migration file **in the same commit** as the model change.
- Destructive migrations (drop column, rename) require two-phase deploy: (1) deploy code that no longer uses the column; (2) deploy the migration that removes it.
- Keep `atomic = True` (default) for PostgreSQL. Use `atomic = False` only for MySQL DDL+DML combos.
- Squash migrations periodically within an app using `squashmigrations` after all environments are current.

---
