---
layout: post_page
title: Using database columns as the golden source of constraints with Clojure
---

I've recently been working on data input on a project using Clojure and PostgreSQL. The database columns were
created, specifying maximum column sizes on columns such as `username` and `password`:

{% highlight sql %}
    CREATE TABLE users
    (
      id         SERIAL PRIMARY KEY,
      username   VARCHAR(50)  NOT NULL UNIQUE,
      email      VARCHAR(100) NOT NULL UNIQUE,
      -- more columns here....
    )
{% endhighlight %}

It's been a while since I've used a SQL database in anger, and there is something very reassuring about
having a reliable backing store that will not let any bad data in. However, I found
that I had these max-length properties of `50` for username and `100` for email in several places:

* the database DDL scripts
* controllers, used to validate on the server side
* the login HTML page, setting `maxlength` attributes on `input` tags
* the registration page
* the forgotten-password page

After a bug was found that was caused by the value being wrong in one of those 5 places, I decided that there
must be a better approach.

The approach in a nutshell
--------------------------

The database is the best golden source for something like max-column lengths, because that's where the data lives.
By enforcing in the database, it doesn't matter how bug-ridden your software is, no bad data will get in. So, the
approach is this:

1. On application startup, load column meta data from the database for the current schema
2. Create a mapping of tables and columns to their various constraint values
3. Query this mapping from code to get values, and associate the map with the HTML templating engine for use in HTML.

Querying the data and creating the map
--------------------------------------

The following SQL will give you lots of information on the columns in your database for a schema named `public` (change
this to whatever your schema is), and although tested only in PostgreSQL should work in most database servers:

{% highlight sql %}
    SELECT * FROM INFORMATION_SCHEMA.COLUMNS
    WHERE table_schema = 'public'
{% endhighlight %}

Of interest for max column lengths are the columns `table_name`, `column_name`, and `character_maximum_length`. By querying
just those 3 columns, it's quite trivial to build up a map in clojure containing all the tables and columns in your database:

{% highlight clojure %}
    (def constraints

        ; load constraints from DB into cols
        (let [cols (db/query "SELECT table_name, column_name, 
                              character_maximum_length
                              FROM INFORMATION_SCHEMA.COLUMNS
                              WHERE table_schema = 'public'")]
            
             ; convert the result-set into a map
             (reduce (fn [map val]
                         (assoc-in map 
                            [(keyword (val :table_name))
                            (keyword (val :column_name))
                            :max-length]
                            (:character_maximum_length val)))
                    {} cols)))
{% endhighlight %}

Note that this assumes there is a DB function called `query` that returns a list of column-value pairs for a SQL query.
All this does is query the column info, then add values keyed by table name, column name, and constraint type. Here is a
test showing example usage from Clojure:

{% highlight clojure %}

(deftest constraints-map
    (testing "maximum lengths can be accessed via the big map of constraints"
        (is (= 50 (get-in constraints [ :users :username  :max-length])))))
{% endhighlight %}

Of course, you might want to make it a bit easier:

{% highlight clojure %}

(defn max-length [table-key column-key]
    (get-in constraints [table-key column-key :max-length]))
      
(testing "the maximum length of the username field is known"
    (is (= 50 (max-length :users :username))))
{% endhighlight %}  

To use this from an HTML templating engine, one approach is to simply associate the whole `constraints` map with your
templating library. For example, you may have a place where some default values are assigned to the model, in which case
you can just associate `:constraints constraints` and then use it from any HTML page. For example, the following sets
the `maxlength` attribute on the login page:

{% highlight html %}
<input type="text" name="username" 
       maxlength="{% raw %}{{constraints.users.username.max-length}}{% endraw %}">
{% endhighlight %}

Now if we need to increase or decrease the length of something like the username, we just change the column length in
the database and all the validation in the HTML and server-side code will be updated.

Taking it further
-----------------

It would also be possible to check if columns were nullable or not, and use this to figure out if fields are required
or not. More interesting is the fact that `check constraint` values in many DBs support regexes. You could extract the
`check constraint` value from the DB, and then associate it with the `pattern` attribute on `input` fields so that you
don't have duplicate regexes throughout your code.