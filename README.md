Importing/Indexing database (MySQL or SQL Server) in Solr using Data Import Handler
Install Solr
download and install Solr from http://lucene.apache.org/solr/.

you can access Solr admin from your browser: http://localhost:8983/solr/

use the port number used in installation.

MySQL connector
Download JDBC driver for MySQL from http://dev.mysql.com/downloads/connector/j/.

Copy file from the downloaded archive 'mysql-connector-java-*.jar' to the folder 'contrib/dataimporthandler/lib' in the folder where Solr was installed. Create 'lib' folder if needed.

MS SQL Server connector
Download Microsoft JDBC Driver 4.0 for SQL Server from: http://www.microsoft.com/en-us/download/details.aspx?displaylang=en&id=11774

copy file 'sqljdbc4.jar' to 'contrib/dataimporthandler/lib'

Setup a new collection
create a new folder for a new collection - 'myproducts'. The collection will be located in '/solr/myproducts' folder. Create folders conf and data in the collection folder:

/solr/myproducts/conf
/solr/myproducts/data
solrconfig.xml
copy solrconfig.xml from an existing collection. Find my version of solrconfig.xml below in this gist.

edit solrconfig.xml by adding:

<lib dir="../../contrib/dataimporthandler/lib" regex=".*\.jar" />
<lib dir="../../dist/" regex="solr-dataimporthandler-.*\.jar" />
Make sure that 'dist' folder contains two files for data import handler:

solr-dataimporthandler-4.10.2.jar
solr-dataimporthandler-extras-4.10.2.jar
add these lines to solrconfig.xml:

<requestHandler name="/dataimport" class="org.apache.solr.handler.dataimport.DataImportHandler">
    <lst name="defaults">
    <str name="config">data-config.xml</str>
    </lst>
</requestHandler>
data-config.xml for MySQL database
the file 'data-config.xml' will define data we want to import/index from our datasource. Assuming that our DB named mydb1 and we have table products with columns id, name and updated_at. Column 'updated_at' of datetime type stores the date of last modification of the row. This column will be used in incremental import to track rows modified since the last import into Solr.

# define data source
<dataConfig>
<dataSource type="JdbcDataSource" 
            driver="com.mysql.jdbc.Driver"
            url="jdbc:mysql://localhost:3306/mydb1" 
            user="root" 
            password=""/>
<document>
  <entity name="product"  
    pk="id"
    query="select id,name from products"
    deltaImportQuery="SELECT id,name from products WHERE id='${dih.delta.id}'"
    deltaQuery="SELECT id FROM products  WHERE updated_at > '${dih.last_index_time}'"
    >
     <field column="id" name="id"/>
     <field column="name" name="name"/>       
  </entity>
</document>
</dataConfig>
The 'query' gives the data needed to populate fields of the Solr document in full-import
The 'deltaImportQuery' gives the data needed to populate fields when running a delta-import
The 'deltaQuery' gives the primary keys of the current entity which have changes since the last index time
Full-import command uses the "query" query, delta-import command uses the delta components.

data-config.xml for SQL Server database
<dataConfig>
  <dataSource type="JdbcDataSource" 
              driver="com.microsoft.sqlserver.jdbc.SQLServerDriver" 
              url="jdbc:sqlserver://servername\instancename;databaseName=mydb"   
              user="sa" 
              password="mypass"/>
  <document>
    <entity name="product"  
      pk="id"
      query="select id,name from products"
      deltaImportQuery="SELECT id,name from products WHERE id='${dih.delta.id}'"
      deltaQuery="SELECT id FROM products  WHERE updated_at > '${dih.last_index_time}'"
      >
       <field column="id" name="id"/>
       <field column="name" name="name"/>       
    </entity>
  </document>
</dataConfig>
schema.xml
edit file 'schema.xml' accordingly to fields defined in data-import.xml:

<schema name="example" version="1.5">
    <field name="_version_" type="long" indexed="true" stored="true"/>
    <field name="id" type="string" indexed="true" stored="true" required="true" multiValued="false" /> 
    <field name="name" type="text_general" indexed="true" stored="true"/>
    ...
add collection to solr
use admin interface in your browser to add a new collection. Add core:

name: myproducts
instanceDir: myproducts
Perform full or incremental import
After successfully adding a collection to Solr you can select it and run dataimport commands:

full-import - use URL: http://localhost:8983/solr/myproducts/dataimport?command=full-import
delta-import - use URL: http://localhost:8983/solr/myproducts/dataimport?command=delta-import
The full import loads all data every time, while incremental import means only adding the data that changed since the last indexing. By default, full import starts with removal the existing index (parameter clean=true).

Note! Use clean=false while running delta-import command.

debug=true - The debug mode limits the number of rows to 10 by default and it also forces indexing to be synchronous with the request.

References
http://wiki.apache.org/solr/DIHQuickStart
http://wiki.apache.org/solr/DataImportHandler
