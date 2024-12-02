# eloquent

Eloquent is a semantic data modelling language designed by and for the age of LLMs. It is a simple, human-readable way to describe the relationships between concepts in a way that can be easily understood by both humans and machines. 

Research has shown that generative AI models have difficulty performing SQL joins (especially multi-hop) and so prefer to work with few or just one large table. This is a problem for data engineers who must then anticipate all possible queries and create scripts that can answer them. Eloquent generates many millions of such queries and scripts automatically. 

Traditional data architecture is grounded in the data to the exclusion of all else. The documentation is a list of tables. It says we should only understand the business to the extent we understand its data.

Semantic data architecture is reversed; it finds grounding in the natural language of the business itself. The documentation is a dictionary of business terms and relevant context. It says we should first understand the whole business and then try to use what data we have.

Eloquent uses the same documentation to model a complex business domain as physical and logical simultaneously, with all the required transformations between them.

Here's an example of a simple Order:

```
**Person**: A human being
- Customer is a Person

**Customer**: A person or organization that buys products
- Customer has a Customer Zipcode

**Customer Zipcode**: The postal code of the customer
- Customer Zipcode is a String

**Order**: A record of a customer's purchase of one or more products
- Order has a Customer
- Order must have an Order Date

**Order Date**: The date the order was placed
- Order Date is a Date
```

At first glance its just a typical data dictionary in Markdown similar to notes that a data architect might take. Easy to edit and manage through source control. But is also encodes information neccessary to understand the inheritance, composition, metrics, and aggregation language used by the business as well as the physical properties of the underlying concepts.

You'll notice a certain rhythm to the Eloquent language. These kinds of repetitions and patterns help an LLM form and organize connections between the concepts. They also provide a way of directly encoding the nodes and edges of the metadata graph that the LLM will use to navigate the data.

Lets see all the different ways we can leverage this very simple model to build out our semantic data architecture. 

- Understand the physical layout of the data for a particular concept. Note that `person_id` and `customer_id` are not defined in the dictionary, but they are implied by Customer's inheritance from Person and used to represent the entity in the physical layer. You can also explicitly define a term as an identifier to use it instead.
```bash
> eloquent describe physical Customer
Customer
- person_id: String
- customer_id: String
- customer_zipcode: String
```

- Write a CREATE TABLE statement that could be filled with data that matches the physical model
```bash
> eloquent create physical Customer
CREATE TABLE physical_customer (
  person_id VARCHAR(255),
  customer_id VARCHAR(255),
  customer_zipcode VARCHAR(255)
);
```

- Understand the logical model using the physical model and the relationships between other models to inherit additional attribues and metrics via `has a` and `is a` type relationships.
```bash
> eloquent describe logical Order
Order
- Order Date
- Customer
- Customer Zipcode
- Person
```


- Create a virtual table that applies the appropriate joins and filters to create the dataset behind the logical model. Note that `order_id` is an identifier and but is being included in the logical model because it functions as the primary key. Also note that the logical 
```bash
> eloquent create logical Order
CREATE VIEW logical_order AS
SELECT
  order_id,
  order_date,
  customer_id,
  customer_zipcode,
  person_id
FROM physical_order
JOIN customer ON order.customer_id = customer.customer_id
JOIN person ON customer.person_id = person.person_id;
```


- If the concepts inherits relationships from another concept, the view will also have the logic map those values and potentially apply some summarization or filtering. Inheritance always accumulates primary key identifiers.
```bash
> eloquent create logical Customer
CREATE VIEW logical_customer AS
WITH person AS (
  SELECT
    person_id
  FROM logical_person
)
SELECT
  physical_customer.person_id,
  physical_customer.customer_id,
  physical_customer.customer_zipcode
FROM physical_customer
JOIN person ON physical_customer.person_id = person.person_id;
```


- Create sql test scripts that can be used to validate the physical and logical models. Each select statement will return no rows or a single row with value 0 if the model is correct and a non-zero number of rows of single row value > 1 if there are errors. Most distributed data technologies do not enforce primary or foreign key relationships so these tests are neccessary to ensure the data is correct.
```bash
> eloquent test logical Order
/* Assert at least one row exists in the logical Order view */ SELECT CASE WHEN COUNT(*) > 0 THEN 0 ELSE 1 END FROM logical_order;
/* Assert no null values for the primary key (order_id) */ SELECT COUNT(*) FROM logical_order WHERE order_id IS NULL;
/* Assert no duplicate values for the primary key (order_id) */ SELECT COUNT(*) FROM logical_order GROUP BY order_id HAVING COUNT(*) > 1;
/* Assert no null values for the required column (order_date) */ SELECT COUNT(*) FROM logical_order WHERE order_date IS NULL;
/* Assert no future order_date values */ SELECT COUNT(*) FROM logical_order WHERE order_date > CURRENT_DATE;
/* Assert no orphaned orders (every order must have a valid person_id) */ SELECT COUNT(*) FROM logical_order WHERE person_id NOT IN (SELECT person_id FROM physical_customer);
/* Assert no null values for the person_id in the Order view */ SELECT COUNT(*) FROM logical_order WHERE person_id IS NULL;
```








**Customer:** A person or organization that buys products
- Customer has a Customer Zipcode
**Customer Zipcode:** The postal code of the customer
- Customer Zipcode is a String
```

A table for the Customer entity would have a customer id primary key and also an optional customer zipcode.


```bash
> eloquent create physical Customer
CREATE TABLE physical_customer (
  person_id VARCHAR(255) PRIMARY KEY,
  customer_zipcode VARCHAR(255)
);
```

A table for the Customer entity would have a customer id primary key and also an optional customer zipcode.

```bash
> eloquent create physical Order
CREATE TABLE physical_order (
  order_id VARCHAR(255) PRIMARY KEY,
  order_date DATE NOT NULL
);
```

A table for the Order entity would have an order id primary key and also an order date. The order date is not null because its described as a must have.

```bash
> eloquent create logical Order
CREATE VIEW order AS
SELECT
  order_id,
  order_date,
  person_id,
  customer_zipcode
FROM order
JOIN customer ON order.person_id = customer.person_id;
```

A logical view of the Order entity would have the same physical columns, but also uses the graph to automatically join to and inherit from many other entities. 

Any term defined in the data dictionary can be used in this way.

```bash 












Given the fixed context window size, these tables should that is lareg enough small enough to fit into the context window. 

Joins are a level of that perform data analysis tasks, either generalists like the GPT-family or specialists like Text2SQL



A Semantic Data Architecture

The architecture itself is provided in a few deliverables:

A model source text document that defines each of the terms used in the model and characterizes the relationships between them. This file is semi-formatted and is intended to be maintained by a data architect using an AI-enhanced text editor or a large language model.

A data dictionary text document which lists each of the terms in the model and all of the first-order relationships to other terms, including those specified in the model source document and any implied or inferred relationships derived from them. It is intended as a reference for data consumers to understand the terminology used in the model and how it can be navigated, as well as RAG-based large language models that need to understand the model.

A visual diagram that shows the relationships between the terms in the model. This diagram is intended to be used by data engineers and others who will work directly with the data to understand the relationships between the terms in the model and how it can be navigated. It is generated algorithmically, so it may change substantially each time the source model is updated.

A set of sql scripts and python orchestration engine that creates and validate the physical views in a database that implements the model. This script is intended to be used by data engineers to create the database schema that will store the data. Silver views are semi-materialized with sample data and Gold views are built entirely from the Silver views. Implementation of the model will be done by replacing the semi-materiliazed Silver views with tables and views that connect directly to your Bronze layer.






