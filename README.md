
Description
-----------
This module is for auto generating Sequelize model from PostgreSQL databases. It reverse engineers your database and generates separate model files for each table. Default configuration parameters are carefully selected and it is sane to use. However it is possible to change most behaviour via configuration. This document is based on default configuration. Configuration parameters and default values are written as (Config parameter:value) where appropriate.

Special Thanks
--------------
This module is developed with sponsorship of Fortibase.

Usage
-----

### Step 1: Install globally via npm -g

This will install module and CLI command

    $ npm install -g sequelize-pg-generator

### Step 2: Generate model files

Open terminal, go to your app.js root and create your models automatically into 'model' directory.

    $ cd path/to/my/node-app.js
    $ spgen -database my_database -u my_user -p my_password

### Step 3: Use it in your node.js app

Use Sequelize models provided by auto generated files in your application.

    var orm = require('../model');
    orm.setup('template_sequelize', 'my_user', 'my_password', {
        host: '127.0.0.1',
        logging: false,
        native: true
    });
    var sequelize = orm.sequelize;
    var contact = orm.model('public.contact'); // Can be configured without schema.

CLI Options
-----------
    spgen [options]
    -h, --host [host]           IP address or host name of the database server
    -pr, --port [port]          Port of database server to connect
    -d, --database [database]   Database name
    -u, --user [user]           Username to connect to database
    -p, --password [password]   Password to connect to database
    -s, --schema [schema]       Comma separated names of the database schemas
    -o, --output [output]       Output folder
    -c, --config [config]       Path of the configuration file

* Multi schema support,
* One to many relation support, (hasMany and belongsTo)
* Many to many relation support, (hasMany through and belongsToMany)
* Inter-schema relation support. (i.e. public.account table to other_schema.cutomer table),
* Highly configurable,
* Fully customizable,
* CLI support,
* Smart naming of models and relations,
* Very easy to override auto generated files
* exclude tables

WARNING: belongsToMany
----------------------
For many to many relations Sequelize version 2.0 RC3 and older does not support belongsToMany relation. After this version hasMany through relations are deprecated. Behaviour on this subject can be adjusted via configuration. (Config: generate.hasManyThrough:false and generate.belongsToMany:true)

Features
--------

### Multi Schema Support
Supports multi PostgreSQL schemas. It is possible to have other schemas than public. User can select which schemas to reverse engineer via CLI or config. If more than one schema is included, models may be prefixed with schema names. (Config: generate.useSchemaName: true, database.schema: ["public"]) to prevent tables with same name in different schemas try to have same model name.

    contact = orm.model('public.contact'); // Returns sequelize model for contact table.
    
### No Schema Prefix
User can configure not to prefix model names with schema
    
    // In config
    {...
        generate{
            useSchemaName: false
        }
    }
    
    contact = orm.model('contact'); // Returns sequelize model for contact table.
    
### One To Many Relation Support
This module automatically detects one to many relations and generates model.hasMany and model.belongsTo sequelize relations.

### Many To Many Relation Support
If two tables are joined via a join table this module detects it automatically and generates many to many sequelize relations. Detection method is as follows: If a table has more than one foreign key, then it is considered

                      hasMany              hasMany
    TABLE:   product --------< line_item >--------- cart
    COLUMNS: id                cart_id              id
             name              product_id           customer_id
             color             quantity

This module generates belongsToMany relation and hasMany relation with through option. As of this writing master branch of Sequelize deprecated hasMany through relations. According to version of Sequelize you use, it should be adjusted via config of this module.

### Inter-Schema Relation Support
Detects relations between tables in different schemas. For example relations between public.account table and other_schema.customer table.

### Highly configurable
This module uses node-configure module. It is also possible to point a custom configuration file via CLI. See configuration parameters below in this document.

### Fully Customizable
This module uses [consolidate](https://www.npmjs.com/package/consolidate) compatible templates to generate model files. It uses [Swig](https://www.npmjs.com/package/swig) by default. User can use his/her custom templates without altering original one by pointing (COnfig: template.folder and template.engine:'swig') config values. Looking default templates in template folder of this module is highly recommended.

There should at least be three files in custom template folder:
    index.ext  Default template file. ext is whatever extension is used for your template engine.
    index.js   This file is copied with generated files. It's purpose is use generated files
    utils.js   This file is copied with generated files. Contains helper functions.

### CLI Supoport
If this module is installed globally with npm -g then sqgen command would be available system wide to generate model files.

### Smart Naming of Models and Relations
pg-sequelize-generator uses table names or schema.table names for model naming. For relations it uses foreign key names and relation names from your database. (You are naming your relations in database meaningfully right?) Both camel case (tableName) or untouched names (table_name) methods can be used via configuration. Naming conventions are based on Sequelize module suggestions and generated explicitly with 'as' parameter.

                      product_cart_line_items              cart_cart_line_items
    TABLE:   product -------------------------< line_item >--------------------- cart
    COLUMNS: id                                 cart_id                          id
             name                               product                          customer_id
             color                              quantity

    NOTE: Beware line_item.cart_id has id suffix but line_item.product hasn't. This inconsistency is made purposefully for the sake of this example.

    Type of object          Naming Rule
    --------------          -----------
    Model                   tableName or schema.tableName
    hasMany                 Plural of the relation name in database. Table name from beginning can be stripped.
                            (Config: generate.stripFirstTableFromHasMany:true)
    belongsTo               Singular of foreign key. If kay name ends with _id it will be stripped. Otherwise
                            'related' is added at the beginning to prevent it gets clash with column name.
                            (Config: generate.prefixForBelongsTo)
    belongsToMany           Plural of the foreign key which refers other table in join table.
    hasMany({through:..})   Plural of the foreign key which refers other table in join table. (DEPRECATED in Sequelize)

    For the example structure:

    Relation                as                          Details
    --------                --                          --
    product.hasMany         as:'cartLineItems'          (Plural) Table name 'product' is stripped from the beginning of
                                                        relation name 'product_cart_line_items'
    product.belongsToMany   as:'carts'                  (Plural) _id suffix is stripped from foreign key name 'cart_id'
    product.hasMany Through as:'carts'                  (Plural) _id suffix is stripped from foreign key name 'cart_id'
    cart.hasMany            as:'cartLineItems'          (Plural) Table name 'cart' is stripped from the beginning of
                                                        relation name 'cart_cart_line_items'
    cart.belongsToMany      as:'relatedProducts'        (Plural) No _id suffix. related is added as prefix.
    cart.hasMany Through    as:'relatedProducts'        (Plural) No _id suffix. related is added as prefix.
    lineItem.belongsTo      as:'relatedProduct'         (Singular) No _id suffix. related is added as prefix.
    lineItem.belongsTo      as:'cart'                   (Singular) No _id suffix. related is added as prefix.

Of course as all attributes, you can modify names of generated files in a non-destructive way as explained below.

### Very Easy to Override Auto Generated Files
By default auto generated files are located path/to/model/definition-files directory. Also there is 'definition-files-custom' directory. Users can create files with same names as auto generated files to override its attributes. There is also utils module generated to make modifications easier.

Those modifications are non destructive, because they override generated file in another file by inheriting it and default index.js file uses inherited files if it exists. Please bear in mind, those modifications occur before sequelize instances are generated.

For example for product table 'definition-files/cart.js' is generated. User can create 'definition-files-custom/cart.js' and override necessary parts like example below. For all attributes you can look inside auto generated files.

    "use strict";
    var orm     = require('../index.js'),
        model   = require('../definition-files/public_cart.js'),
        util    = require('../utils.js')(model),
        Seq     = orm.Sequelize();

    module.exports = model;

    util.getAttribute('id').validate = {... Some Sequelize Validations}; // Add Sequelize validation.
    util.getRelation('relatedProducts').details.as = 'soldItems';        // Don't like default relation name? Change it.
    util.renameAttribute('customerId', 'clientId');                      // Change name of the attribute.

### Exclude Tables

It is possible to exclude some table from auto generation. (Config generate.skipTable:[]) array is used to define excluded tables. sequelize-pg-generator skips those tables and relations from and to those tables.

### Debug

Default index.js file creates a file called debug.js in the model directory. This file can be examined what type of code is used by index.js. It is same code that would be used if there is no index.js file exists. However if this type of static file would be used, it is harder to allow modifications in a non-destructive way.

### Table Specific Configuration

Sometimes for some tables it is needed to have different rules then other tables have. In such situations configuration file allows table level overrides. All 'generate' and 'tableOptions' config parameters can be overridden with 'generateOverride' and 'tableOptionsOverride'.

    "generate": {
        "columnDescription": true,
        "tableDescription": true,
        ...
    },
    "generateOverride": {
        "contact": {
            "tableDescription": false
        }
    },
    "tableOptions": {
        "timestamps": false,
        "camelCase": true,
        "paranoid": false,
        ...
    },
    "tableOptionsOverride": {
        "contact": {
            "paranoid": true
        }
    }
    ...

API
===

#Index

**Modules**

* [lib/index](#module_lib/index)
  * [module.exports(callback, options) ⏏](#exp_module_lib/index)

**Classes**

* [class: GeneratorUtil](#GeneratorUtil)
  * [new GeneratorUtil(model)](#new_GeneratorUtil)
  * [generatorUtil.getRelation(as)](#GeneratorUtil#getRelation)
  * [generatorUtil.getAttribute(name)](#GeneratorUtil#getAttribute)
  * [generatorUtil.renameAttribute(oldName, newName)](#GeneratorUtil#renameAttribute)
 
<a name="module_lib/index"></a>
#lib/index
**Author**: Özüm Eldoğan  
<a name="exp_module_lib/index"></a>
##module.exports(callback, options) ⏏
Generates model files for Sequelize ORM.

**Params**

- callback `function` - Function to execute after completion of auto generation. callback(err)  
- options `object` - Options to override configuration parameters from config file  
  - host `string` - IP address or host name of the database server  
  - port `number` - Port of database server to connect  
  - database `string` - Database name  
  - user `string` - Username to connect to database  
  - password `string` - Password to connect to database  
  - schema `Array` - List of comma separated names of the database schemas to traverse. Example public,extra_schema.  
  - output `string` - Output folder  
  - config `string` - Path of the configuration file  

<a name="GeneratorUtil"></a>
#class: GeneratorUtil
**Members**

* [class: GeneratorUtil](#GeneratorUtil)
  * [new GeneratorUtil(model)](#new_GeneratorUtil)
  * [generatorUtil.getRelation(as)](#GeneratorUtil#getRelation)
  * [generatorUtil.getAttribute(name)](#GeneratorUtil#getAttribute)
  * [generatorUtil.renameAttribute(oldName, newName)](#GeneratorUtil#renameAttribute)

<a name="new_GeneratorUtil"></a>
##new GeneratorUtil(model)
**Params**

- model   

<a name="GeneratorUtil#getRelation"></a>
##generatorUtil.getRelation(as)
Searches and returns relation with the given alias. Alias is defined in sequelize options with parameter 'as'

**Params**

- as `string` - Alias of the relation.  

**Returns**: `Object`  
<a name="GeneratorUtil#getAttribute"></a>
##generatorUtil.getAttribute(name)
Searches and returns relation with the given attribute. Alias is defined in sequelize options with parameter 'as'

**Params**

- name `string` - Name of the attribute.  

**Returns**: `Object`  
<a name="GeneratorUtil#renameAttribute"></a>
##generatorUtil.renameAttribute(oldName, newName)
Searches and returns relation with the given attribute. Alias is defined in sequelize options with parameter 'as'

**Params**

- oldName `string` - Name of the attribute which it's name to be changed.  
- newName `string` - New name of the attribute.  


---------------------------------------

<a name="History"></a>
History & Release Notes
=======================

Note
----
Version history for minimal documentation updates are not listed here to prevent cluttering.
Important documentation changes are included anyway.

0.0.1 / 2014-12-23
==================
Initial version