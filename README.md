This example shows a way to connect a Ruby on Rails web application with julia through ZMQ.

Basically we create a ZMQ server in julia that will perform some predefined calculation as instructed from a web page. In this case, supply a number and *magically* multiply it by 3.

This example was inspired by [a julia-users topic](https://groups.google.com/forum/#!searchin/julia-users/zmq|sort:date/julia-users/umHiBwVLQ4g/8O-bC7UB2B0J), IJulia and the excellent [samurails blog post on delayed job](http://samurails.com/gems/delayed-job/). Parts of the code below is taken from that blog.

## installation

I write this on Ubuntue 14.04. If you are using another platform, you will have to google some installation procedures yourself.

To install Ruby on Rails, the easiest is using [rvm](http://rvm.io):

	curl -sSL https://get.rvm.io | bash -s stable --rails

I prefer working with a PostgreSQL database, but I assume this example to work fine with the standard MySQL. Install PostgreSQL on ubuntu (I know libpq-dev is necessary, the others I don't know):

	sudo apt-get install postgresql-client pgadmin3 postgresql postgresql-contrib libpq-dev

Of course you need julia installed. I used v0.3 for this example. See details on julialang.org. In julia you need to add the ZMQ package by simple `Pkg.add("ZMQ")`.

Once you download this example you should install the necessary gems listed in the gem file:

	git clone https://github.com/Ken-B/RoR_julia_eg
	cd RoR_julia_eg
	bundle install

## follow along

I'll try to commit the outcome of each rails command and describe it in the commit message. Here's what I did.

First create a new rails app called `triple`, that will magically triple each input number you give it:

	rails new triple --database=postgresql

Include the necessary gems in the gemfile

	# Gemfile
	gem 'simple_form' #for the number input on the web page
	gem 'ffi-rzmq'	#for ZMQ client on Rails side
	gem 'delayed_job_active_record' #longer calculations need to be run in background
	gem 'daemons' #for bin/delayed_job start

then install the dependencies:
	
	cd triple
	bundle install

In case you don't have nodejs installed:

	sudo apt-get install nodejs

Initiate your database with

	rake db:create

You need to configure `simple_form` with:

	rails generate simple_form:install

For delayed_job you need to generate the table to store the job queue and actually execute the migration in the database:

	rails generate delayed_job:active_record
	rake db:migrate

Now if you start-up the server with `rails s` there's a welcome screen at (http://localhost:3000/).

---
Now let's create the app. 
We start by adding a controller.

	rails g controller Triples index new

and we set the root for our app:

	# triple/config/routes.rb

For the database we will use a `number` model with a `value` and `result` field for the input and output (= 3*input) fields, as well as a field `calculated` to indicate if the calculation has finished

	rails g model number value:float result:float calculated:boolean

In the migration file that was created we add a `false` default value to the calcaluted field. That line in `triple/db/migrate/20150209001428_create_numbers.rb` now becomes:

	t.boolean :calculated, default: false

Execute the migration in the database with the usual:

	rake db:migrate