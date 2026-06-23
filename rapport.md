# Rapport - Labo 03 : REST APIs, GraphQL

**Étudiant :** Thomas Journault  
**Cours :** LOG430 - Architecture logicielle  

---

## Question 1

Dans la [RFC 7231](https://www.rfc-editor.org/rfc/rfc7231#section-4.2.1), nous trouvons que certaines méthodes HTTP sont considérées comme sûres (__safe__) ou idempotentes, en fonction de leur capacité à modifier (ou non) l'état de l'application. Parmi les méthodes utilisées dans l'activité 2 (`POST /products`, `POST /stocks`, `GET /stocks/:id`, `POST /orders`, `DELETE /orders/:id`), lesquelles sont sûres, non sûres, idempotentes et/ou non idempotentes ?

Selon les sections 4.2.1 et 4.2.2 de la RFC 7231, une méthode est **sûre** si elle ne modifie pas l'état du serveur, et **idempotente** si plusieurs appels identiques produisent le même résultat qu'un seul appel.

Voici l'analyse des méthodes utilisées dans l'activité 2 :

| Méthode | Endpoint | Sûre | Idempotente |
|---|---|---|---|
| `GET` | `GET /stocks/:id` |  Oui |  Oui |
| `POST` | `POST /products`, `POST /stocks`, `POST /orders` |  Non |  Non |
| `DELETE` | `DELETE /orders/:id` |  Non |  Oui |

**`GET /stocks/:id`** — sûre et idempotente. Elle ne fait que lire le stock d'un produit sans modifier l'état de l'application. Appeler cet endpoint plusieurs fois retourne toujours la même réponse.

```python
# store_manager.py
@app.get('/stocks/<int:product_id>')
def get_stocks(product_id):
    return get_stock(product_id)
```

**`POST /products`, `POST /stocks`, `POST /orders`** — pas sûres et pas idempotentes. Chaque appel crée une nouvelle ressource ou modifie l'état. Par exemple, deux appels identiques à `POST /orders` créent deux commandes distinctes et déduisent le stock deux fois.

```python
# store_manager.py
@app.post('/orders')
def post_orders():
    return create_order(request)  # crée une nouvelle commande à chaque appel

@app.post('/products')
def post_products():
    return create_product(request)  # crée un nouveau produit à chaque appel
```

**`DELETE /orders/:id`** — non sûre (elle modifie l'état en supprimant la commande et en remettant le stock), mais idempotente : supprimer une commande déjà supprimée retourne simplement `{"deleted": false}` sans effet secondaire.

```python
# store_manager.py
@app.delete('/orders/<int:order_id>')
def delete_orders_id(order_id):
    return remove_order(order_id)  # idempotent : supprimer deux fois = même résultat
```
Pour un apelle PUT, tout comme DELETE elle serais pas **safe**, mais **idempotente** , car deux fois la même requête ne fera la modification qu'une seul fois sur le stock.


---

## Question 2

Décrivez l'utilisation de la méthode `join` dans le cas de `get_stock_for_all_products`. Utilisez les méthodes telles que décrites à *Simple Relationship Joins* et *Joins to a Target with an ON Clause* dans la documentation SQLAlchemy pour ajouter les colonnes `name`, `sku` et `price`. Veuillez inclure le code pour illustrer votre réponse.

La méthode `join` de SQLAlchemy permet de combiner deux tables en une seule requête, évitant ainsi de faire plusieurs appels séparés à la base de données.



on n'utilise pas *Simple relationship join*, car La documentation SQLAlchemy la décrit comme suit :

```python
# Exemple de la doc — fonctionne quand une relationship() est déclarée
session.query(User).join(User.addresses)
```

Cette syntaxe fonctionne uniquement lorsqu'une `relationship()` est déclarée dans le modèle SQLAlchemy. Dans notre cas, `Stock` et `Product` sont deux modèles indépendants sans `relationship()` définie entre eux :

```python
# src/stocks/models/stock.py
class Stock(Base):
    __tablename__ = 'stocks'
    product_id = Column(Integer, primary_key=True, nullable=False)
    quantity = Column(Integer, nullable=False)
    # Pas de relationship() vers Product

# src/stocks/models/product.py
class Product(Base):
    __tablename__ = 'products'
    id = Column(Integer, primary_key=True, autoincrement=True)
    name = Column(String, nullable=False)
    sku = Column(String, nullable=False)
    price = Column(Float, nullable=False)
```


Sans `relationship()`, on utilise le style *Joins to a Target with an ON Clause* en passant explicitement la condition de jointure. Cela correspond à l'équivalent SQL `JOIN ... ON` :

```python
# src/stocks/queries/read_stock.py
def get_stock_for_all_products():
    session = get_sqlalchemy_session()
    results = session.query(
        Stock.product_id,
        Stock.quantity,
        Product.name,
        Product.sku,
        Product.price,
    ).join(Product, Stock.product_id == Product.id).all()
```

Le `.join(Product, Stock.product_id == Product.id)` indique à SQLAlchemy de faire une jointure entre la table `stocks` et la table `products` sur la condition `stocks.product_id = products.id`. Cela génère la requête SQL suivante :

```sql
SELECT stocks.product_id, stocks.quantity, products.name, products.sku, products.price
FROM stocks JOIN products ON stocks.product_id = products.id
```

Chaque ligne du résultat contient ainsi les colonnes des deux tables, ce qui permet de construire le rapport complet :

```python
    for row in results:
        stock_data.append({
            'Article': row.name,
            'Numéro SKU': row.sku,
            'Prix unitaire': row.price,
            'Unités en stock': int(row.quantity),
        })
```

---

## Question 3

Quels résultats avez-vous obtenus en utilisant l'endpoint `POST /stocks/graphql-query` avec la requête suggérée ? Veuillez joindre la sortie de votre requête dans Postman afin d'illustrer votre réponse.

En envoyant la requête suivante à `POST /stocks/graphql-query` :

```graphql
{
  product(id: "1") {
    id
    quantity
  }
}
```

On obtient la réponse JSON suivante :

```json
{
    "data": {
        "product": {
            "id": 1,
            "quantity": 50
        }
    },
    "errors": null
}
```

![Postman Q3](image/Postmanq3.png)

L'endpoint retourne uniquement les champs `id` et `quantity`, car ce sont les seuls champs définis dans le schéma GraphQL pour l'instant. Les champs `name`, `sku` et `price` ne sont pas encore accessibles sur GraphQL puisqu'ils n'ont pas encore été ajoutés au type `ProductType` ni récupérés depuis Redis. La réponse est cohérente avec la structure du schéma GraphQL défini avant l'activité 5 :

```python
# src/stocks/schemas/product.py
class Product(ObjectType):
    id = Int()
    quantity = Int()
```

Si un champ non défini (comme `name`) est demandé dans la requête, GraphQL retourne une erreur de validation avant même d'exécuter la requête, ce qui illustre l'un des avantages de GraphQL.

---

## Question 4

Quelles lignes avez-vous changé dans `update_stock_redis` (fichier `src/stocks/commands/write_stock.py`) ? Veuillez joindre du code afin d'illustrer votre réponse.

Trois modifications ont été apportées :

**1. Import du modèle `Product`** en haut du fichier :

```python
from stocks.models.product import Product
```

**2. Ouverture d'une session SQLAlchemy** dans la boucle pour pouvoir interroger la BD :

```python
session = get_sqlalchemy_session()
try:
    for item in order_items:
        ...
finally:
    session.close()
```

**3. Récupération des informations du produit et stockage dans Redis via un `mapping`** au lieu d'une seule valeur :

Avant :
```python
pipeline.hset(f"stock:{product_id}", "quantity", new_quantity)
```

Après :
```python
product = session.query(Product).filter_by(id=product_id).first()
mapping = {"quantity": new_quantity}
if product:
    mapping["name"] = product.name
    mapping["sku"] = product.sku
    mapping["price"] = product.price

pipeline.hset(f"stock:{product_id}", mapping=mapping)
```

Avant ces changements, Redis ne stockait que la `quantity` pour chaque clé `stock:{id}`. Après, il stocke aussi `name`, `sku` et `price`, ce qui permet au resolver GraphQL de retourner ces informations sans avoir à refaire de requête SQL à chaque fois.

---

## Question 5

Quels résultats avez-vous obtenus en utilisant l'endpoint `POST /stocks/graphql-query` avec les améliorations de l'activité 5 ? Veuillez joindre la sortie de votre requête dans Postman afin d'illustrer votre réponse.

Après avoir créé une commande de 2 unités du produit 1 (ce qui déclenche `update_stock_redis`), la requête suivante :

```graphql
{
  product(id: "1") {
    id
    name
    sku
    price
    quantity
  }
}
```

retourne :

```json
{
    "data": {
        "product": {
            "id": 1,
            "name": "Laptop ABC",
            "price": 1999.99,
            "quantity": 48,
            "sku": "LP12567"
        }
    },
    "errors": null
}
```

![Postman Q5](image/Postmanq5.png)

Contrairement à la Question 3, les champs `name`, `sku` et `price` sont maintenant retournés. Ces informations sont stockées dans Redis par `update_stock_redis` lors de chaque opération sur les commandes, évitant ainsi une requête SQL supplémentaire au moment de la lecture GraphQL. La quantité est passée de 50 à 48 suite à l'achat de 2 unités.

---

## Question 6

Examinez attentivement le fichier `docker-compose.yml` du répertoire `scripts`, ainsi que celui situé à la racine du projet. Qu'ont-ils en commun ? Par quel mécanisme ces conteneurs peuvent-ils communiquer entre eux ? Veuillez joindre du code YML afin d'illustrer votre réponse.

Les deux fichiers déclarent le même réseau `labo03-network` avec `external: true` :

```yaml
# docker-compose.yml (racine)
networks:
  labo03-network:
    driver: bridge
    external: true

# scripts/docker-compose.yml
networks:
  labo03-network:
    driver: bridge
    external: true
```

Le mot-clé `external: true` indique que le réseau n'est pas créé par Docker Compose, mais qu'il existe déjà sur la machine (créé manuellement avec `docker network create labo03-network`). Les deux compositions se branchent sur ce réseau partagé, ce qui permet à leurs conteneurs de se voir mutuellement.

C'est par ce mécanisme que `supplier_app` (défini dans `scripts/docker-compose.yml`) peut joindre `store_manager` (défini dans `docker-compose.yml` à la racine) en utilisant simplement son nom de service comme hostname :

```yaml
# scripts/supplier_app.py
ENDPOINT_URL = "http://store_manager:5000/stocks/graphql-query"
```

Docker résout automatiquement `store_manager` vers l'adresse IP du conteneur correspondant grâce au `labo03-network`. Sans ce réseau partagé, les deux compositions seraient isolées et `supplier_app` ne pourrait pas contacter `store_manager`.

---

## Annexe 1 — GitHub Actions Runner

![Runner 1](image/runner1.png)

![Runner 2](image/runner2.png)

---

## Annexe 2 — Tests

![Tests](image/test.png)
