# Divergence Framework Documentation

## Table of Contents
- ### [Getting Started](/gettingstarted.md)
    - [Server Prerequisites](/gettingstarted.md#server-prerequisites)
    - [Bootstrap a New Project](/gettingstarted.md#bootstrap-a-new-project)
    - [How to Bootstrap Manually (Advanced)](/gettingstarted.md#how-to-bootstrap-manually-advanced)
    - [Establish Your Classes Directory](/gettingstarted.md#establish-your-classes-directory)
    - [Configure Database Access](/gettingstarted.md#configure-database-access)
    - [Configuring nginx or apache2 Servers](/gettingstarted.md#configuring-nginx-or-apache2-servers)
    
- ### [Command Line Tool](/cli.md)
    - [Installation](/cli.md#installation)
    - [Initializing a Project](/cli.md#initializing-a-project)
    - [Testing Your Database Config](/cli.md#testing-your-database-config)
    - [Change Your Database Config](/cli.md#change-your-database-config)
    - [Tool Usage Reference](/cli.md#tool-usage-reference)

- ### [Project Structure](/projectstructure.md)
    - [Load Order](/projectstructure.md#load-order)
    - [Folder Structure](/projectstructure.md#folder-structure)
    - [The App Class](/projectstructure.md#the-app-class)
    - [Configs](/projectstructure.md#configs)
    - [Development & Production](/projectstructure.md#development---production)
    - [Error Handling](/projectstructure.md#error-handling)

- ### [ORM](/orm.md#orm)
    - [Model Architecture](/orm.md#model-architecture)
    - [Making a Basic Model](/orm.md#making-a-basic-model)
    - [Create, Update, and Delete](/orm.md#create-update-and-delete)
    - [Versioning](/orm.md#versioning)
    - [Relationships](/orm.md#relationships)
    - [Supported Field Types](/orm.md#supported-field-types)
    - [Canary Model - An Example Utilizing Every Field Type](/orm.md#canary-model---an-example-utilizing-every-field-type)
    - [Validation](/orm.md#validation)
    - [Event Binding](/orm.md#event-binding)
    - [Advanced Techniques](/orm.md#advanced-techniques)

- ### [Controllers](/controllers.md#controllers)
    - [Intro to Tree Routing](/controllers.md#intro-to-tree-routing)
    - [RequestHandler](/controllers.md#requesthandler)
    - [Your Own Controllers](/controllers.md#your-own-controller)
    - [RecordsRequestHandler](/controllers.md#recordsrequesthandler)
        - [Building an API](/controllers.md#recordsrequesthandler)
        - [Permissions](/controllers.md#permissions)
        - [JSON API Reference](/controllers.md#json-api-reference)
            - [Browse](/controllers.md#browse) (Read Many)
            - [One Record](/controllers.md#one-record) (Read One)
            - [Edit One Record](/controllers.md#edit-one-record)
            - [Create One Record](/controllers.md#create-one-record)
            - [Delete One Record](/controllers.md#delete-one-record)
            - Create or Edit Multiple Records
            - Delete Multiple Records
    - Advanced Techniques

- ### [Views](/views.md)
    - [Responding with a Template](/views.md#responding-with-a-template)

- ### ~~Security~~
    - ~~User model~~
    - ~~Session model~~
    - ~~Authentication~~
    - ~~Binding Permissions~~

    **Note:** *While these things are already built (and running in production on many many sites) they are yet to be unit tested so for now they remain on the road map for future versions.*

- ### Developing
    - Clone Framework in Development Mode
    - Setting up a unit testing environment
    - Running tests
    - Style Guide

- ### [Addendum](/addendum.md)
    - [Background](/addendum.md#background)
    