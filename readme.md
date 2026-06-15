# Reutilización de `billing` en la app `purchasing`

## Contexto

El proyecto Django (`Sales_A2`) contiene la app **`billing`**, que centraliza los modelos
maestros del negocio: `Brand`, `ProductGroup`, `Supplier`, `Product`, `Customer` y los
documentos de venta (`Invoice` / `InvoiceDetail`).

La futura app **`purchasing`** (órdenes de compra a proveedores) **no duplica** estos
modelos: los importa directamente desde `billing`.

---

## Cómo se reutilizan `Supplier` y `Product`

### 1. Importación directa en `purchasing/models.py`

```python
from billing.models import Supplier, Product

class PurchaseOrder(models.Model):
    supplier = models.ForeignKey(
        Supplier,
        on_delete=models.PROTECT,
        related_name='purchase_orders',
    )
    order_date = models.DateTimeField(auto_now_add=True)
    total = models.DecimalField(max_digits=12, decimal_places=2, default=0)

class PurchaseOrderDetail(models.Model):
    order   = models.ForeignKey(PurchaseOrder, on_delete=models.CASCADE, related_name='details')
    product = models.ForeignKey(Product, on_delete=models.PROTECT, related_name='purchase_details')
    quantity    = models.IntegerField(default=1)
    unit_cost   = models.DecimalField(max_digits=12, decimal_places=2)
```

`Supplier` ya tiene `name`, `contact_name`, `email`, `phone` y `is_active`.  
`Product` ya lleva `unit_price`, `stock`, la FK a `Brand`/`ProductGroup` y la M2M a `Supplier`.

### 2. Registro en `settings.py`

```python
INSTALLED_APPS = [
    ...
    'billing',      # debe ir antes
    'purchasing',   # depende de billing
]
```

### 3. Actualización de stock al recibir una orden

Al confirmar una compra, `purchasing` actualiza el `stock` del `Product` que vive en `billing`:

```python
# purchasing/views.py (simplificado)
product = Product.objects.get(pk=detail.product_id)
product.stock += detail.quantity
product.save()
```

---

## Ventajas del enfoque

| Aspecto | Resultado |
|---|---|
| **Sin duplicación** | `Supplier` y `Product` se mantienen en un solo lugar |
| **Consistencia** | Cambios de precio o stock se reflejan en ambas apps |
| **Migraciones limpias** | `purchasing` no genera tablas para modelos ya existentes |