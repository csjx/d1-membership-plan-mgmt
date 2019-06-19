Membership Plan Management
==========================

.. contents:: Table of Contents
    :depth: 2

Overview
--------

To support ongoing operations, DataONE offers paid services for memberships. This document outlines the design and implementation details needed to offer these services. It describes the Plans, Subscriptions, Products, Customers, Orders, Invoices, Charges, and Quotas that DataONE needs to track. This documents:

- Who has subscribed to Membership Plans
- What Products the Plans include
- What additional Products they add into their Order
- What Invoices have been sent for an Order
- Which payment Charge(s) completed the Order
- What Quota limits are set for Customers per Product.

Details of how the payment will be collected is to be determined, but will involve the UCSB extramural funds payment service. Personally identifiable information that is stored will be limited to names, emails, and billing addresses, but will exclude financial transaction details (credit cards, etc.) other than the outcome of a Charge transaction.

The following diagram shows the membership and payment records stored by DataONE and their relationships.

..
    @startuml images/overview.png
    !include ./plantuml-styles.txt
    'left to right direction
    class Plan {
    }
    class Product {
    }
    class Subscription {
    }
    class Customer {
    }
    class Order {
    }
    class Invoice {
    }
    class Charge {
    }
    class Quota {
    }
    
    Subscription "1" --o "1" Product : "   associated with"
    Plan "1" -left-o "1" Subscription : "associated with"
    Customer "1" o-left- "1" Subscription : "associated with"
    Customer "1" --o "n" Order : "   associated with"
    Order "0" -right-o "n" Product : "associated with"
    Order "1" -up-o "n" Charge : "   associated with"
    Order "1" -left-o "n" Invoice : "   associated with"
    Product "0"--o "n" Quota : "   associated with"
    
    @enduml
    
.. image:: images/overview.png

Plans
-----
A DataONE Plan defines a price, currency, and billing cycle for a given subscription service product.  There are initially four named subscription plans offered:

- Individual
- Organization
- Institution
- Institution+

A Plan is based on a particular Product, identified by the product ID.  The associated Product may change over time in order to add features to the Plan, like increasing the custom portal count, etc.  Arbitrary key-value pair fields can be added to the ``metadata`` field to keep track of internal Plan metadata.  Multiple Plans may be associated with a given Product so you can offer them in different currencies.  The list of active Plans are publicly accessible for clients like MetacatUI or the DataONE website to render.

..
    @startuml images/plan.png
    !include ./plantuml-styles.txt

    class Plan {
        id: string
        object: string
        active: boolean
        amount: integer
        billing_scheme: string
        created: integer
        currency: string
        interval: string
        interval_count: integer
        metadata: hash
        nickname: string
        product: string
        }
    @enduml

.. image:: images/plan.png

An example Plan:

.. code:: json

    {
        "id": "445A35DC-A14D-4D89-8B4F-61118D9B3BEA",
        "object": "plan",
        "active": true,
        "amount": 300000,
        "billing_scheme": "send_invoice", 
        "created": 1559768309,
        "currency": "USD",
        "interval": "year",
        "interval_count": 1,
        "metadata": {},
        "nickname": "Organization",
        "product": "725C2F79-7E0B-4018-94F3-C16D05F23CCC"
    }

Subscriptions
-------------

Subscriptions are products that are billed on a recurring basis, and associate a Customer with a particular Plan.  Subscriptions have creation, start, end and cancel dates used to keep track of the status of the Subscription.  The status field changes based on timely payment of the latest Invoice associated with the Subscription.  Individual Subscriptions can be accessed by the Customer or by an administrator.

..
    @startuml images/subscription.png
    !include ./plantuml-styles.txt

    class Subscription {
        id: string
        object: string
        billing: string
        billing_cycle_anchor: timestamp
        billing_thresholds: hash
        canceled_at: timestamp
        created: timestamp
        current_period_end: timestamp
        current_period_start: timestamp
        customer: string
        days_until_due: integer
        discount: hash
        ended_at: timestamp
        items: array of hashes
        latest_invoice: string
        metadata: hash
        plan: string
        quantity: integer
        start: timestamp
        start_date: timestamp
        status: string
        }
    @enduml

.. image:: images/subscription.png

Products
--------

Products define the exact DataONE service offered, and describe the features of the service using the extensible ``metadata`` field.  Each Product is unique, and can be tied to multiple pricing Plans.  Other Products offered are not tied to Plans, but may be part of any Order, such as training or consultation Products.  DataONE keeps a catalog of Products offered over time which may be listed by client applications.

..
    @startuml images/product.png
    !include ./plantuml-styles.txt

    class Product {
        id: string
        object: string
        active: boolean
        name: string
        caption: string
        description: string
        created: timestamp
        statement_descriptor: string
        type: string
        unit_label: string
        url: string
        metadata: hash
        quotas: list
    }
    @enduml

.. image:: images/product.png

An example Product:

.. code:: json

    {
        "id": "725C2F79-7E0B-4018-94F3-C16D05F23CCC",
        "object": "product",
        "active": true,
        "name": "Organization",
        "caption": "Small institutions or groups",
        "description": "Create multiple portals for your work and projects. Help others understand and access your data.",
        "created": 1559768309,
        "statement_descriptor": "DataONE Membership Plan - Organization",
        "type": "service",
        "unit_label": "membership",
        "url": "https://dataone.org/memberships/organization",
        "metadata": {
            "features": [
                {
                    "name": "custom_portal",
                    "label": "Branded Portals",
                    "description": "Showcase your research, data, results, and usage metrics by building a custom web portal.",
                    "count": 3
                },
                {
                    "name": "custom_search_filters",
                    "label": "Custom Search Filters",
                    "description": "Create custom search filters in your portal to allow scientists to search your holdings using filters appropriate to your field of science."
                },
                {
                    "name": "fair_data_assessment",
                    "label": "FAIR Data Assessments",
                    "description": "Access quality metric reports using the FAIR data suite of checks."
                },
                {
                    "name": "custom_quality_service",
                    "label": "Custom Quality Metrics",
                    "description": "Create a suite of custom quality metadata checks specific to your datasets."
                },
                {
                    "name": "aggregated_metrics",
                    "label": "Aggregated Metrics",
                    "description": "Access and share reports on aggregated usage metrics such as dataset views, data downloads, and dataset citations."
                },
                {
                    "name": "dataone_voting_member",
                    "label": "DataONE Voting Member",
                    "description": "Vote on the direction and priorities at DataONE Community meetings."
                }
            ]
        }
    }

Customers
---------

Customers are associated with a DataONE account (by ORCID), and are associated with Subscriptions, Orders, Invoices, Charges, and Quotas based on certain purchased Products.
 
..
    @startuml images/customer.png
    !include ./plantuml-styles.txt

    class Customer {
        id: string
        object: string
        balance: integer
        address: hash
        created: timestamp
        currency: string
        delinquent: boolean
        description: string
        discount: hash
        email: string
        invoice_prefix: string
        invoice_settings: hash
        metadata: hashes
        name: string
        phone: string
        subscriptions: list
        tax_exempt: string
    }
    @enduml

.. image:: images/customer.png

Quotas
------

Quotas are limits set for a particular product, such as the number of portals allowed, disk space allowed, etc. Quotas have a soft and hard limit per unit to help with communicating limit warnings.

..
    @startuml images/quota.png
    !include ./plantuml-styles.txt

    class Quota {
        id: string
        object: string
        name: string
        soft_limit: integer
        hard_limit: integer
        unit: string
    }
    @enduml

.. image:: images/quota.png

Orders
------

Orders track Customer purchases of a list of Products, and the total amount of the Order that was charged in a Charge.

..
    @startuml images/order.png
    !include ./plantuml-styles.txt

    class Order {
        id: string
        object: string
        amount: integer
        amount_returned: integer
        charge: string
        created: timestamp
        currency: string
        customer: string
        email: string
        items: array of hashes
        metadata: hash
        status: string
        status_transitions: hash
        updated: timestamp
    }
    @enduml

.. image:: images/order.png

Charges
-------

Charges document transactions against a given payment source, like a credit card.  While DataONE won't track payment sources, we will track Charge events by ID as part of an Order.

..
    @startuml images/charge.png
    !include ./plantuml-styles.txt

    class Charge {
        id: string
        object: string
        amount: integer
        amount_refunded: integer
        created: timestamp
        currency: string
        customer: string
        description: string
        failure_code: string
        invoice: string
        metadata: hash
        order: string
        outcome: string
        paid: boolean
        receipt_email: string
        refunded: boolean
        refunds: list
        status: string
    }
    @enduml

.. image:: images/charge.png

