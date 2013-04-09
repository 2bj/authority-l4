# Authority-L4
## A simple and flexible authorization system for Laravel 4

### Installation via Composer
Add Authority to your composer.json file to require Authority

	require : {
		"laravel/framework": "4.0.*",
        "machuga/authority-l4" : "dev-master"
    }

Now update Composer

	composer update

The last **required** step is to add the service provider to `app/config/app.php`
	
	'Authority\AuthorityL4\AuthorityL4ServiceProvider',

Congratulations, you have successfully installed Authority.  However, we have also included some other configuration options for your convenience.



### Additional (optional) Configuration Options

##### Add the alias (facade) to your Laravel app config file.

	'Authority' => 'Authority\AuthorityL4\Facades\Authority',

This will allow you to access the Authority class through the static interface you are used to with Laravel components.

	Authority::can('update', 'SomeModel');

##### Publish the Authority default configuration file

	php artisan config:publish machuga/authority-l4

This will place a copy of the configuration file at `app/config/packages/machuga/authority-l4`.  The config file includes an 'initialize' function, which is a great place to setup your rules and aliases.

	//app/config/packages/machuga/authority-l4

	return array(

		'initialize' => function($authority, $user)
		{
			//action aliases
			$authority->addAlias('manage', array('create', 'read', 'update', 'delete'));
        	$authority->addAlias('moderate', array('read', 'update', 'delete'));

        	//an example using the `hasRole` function, see below examples for more details
        	if($user->hasRole('admin'))
        	{
        		$authority->allow('manage', 'all');
			}
		}

	);

##### Create Roles and Permissions Tables

We have provided a basic table structure to get you started in creating your roles and permissions.

Run the Authority migrations

	php artisan migrate --package="machuga/authority-l4"

This will create the following tables

- roles
- role_user
- permissions

To utilize these tables, you can add the following methods to your `User` model.  You will also need to create Role and Permission Model stubs.

	//app/models/User.php

	...

	public function roles()
    {
        return $this->belongsToMany('Role');
    }

    public function permissions()
    {
        return $this->hasMany('Permission');
    }

	public function hasRole($key)
	{
		foreach($this->roles as $role)
		{
			if($role->name == $key)
			{
				return true;
			}
		}
		return false;
	}

	//app/models/Role.php
	<?php
	class Role extends Eloquent {}

	//app/models/Permission.php
	class Permission extends Eloquent {}

Lastly, in your Authority config file which you copied over in the previous configuration step.  You can add some rules:

	<?php
	//app/config/packages/machuga/authority-l4

	return array(

		'initialize' => function($authority, $user)
		{
			//action aliases
			$authority->addAlias('manage', array('create', 'read', 'update', 'delete'));
        	$authority->addAlias('moderate', array('read', 'update', 'delete'));

        	//an example using the `hasRole` function, see below examples for more details
        	if($user->hasRole('admin'))
        	{
        		$authority->allow('manage', 'all');
			}

			// loop through each of the users permissions, and create rules
			foreach($user->permissions as $perm)
			{
				if($perm->type == 'allow')
				{
					$authority->allow($perm->action, $perm->resource);
				}
				else
				{
					$authority->deny($perm->action, $perm->resource);
				}
			}
		}

	);

## General Usage
	
	//If you added the alias to `app/config/app.php` then you can access Authority, from any Model, Controller, View, or anywhere else in your Laravel app like so:
	if( Authority::can('create', 'User') )
	{
		User::Create(array(
			'username' => 'someuser@test.com'
		));	
	}

	//If you just chose to use the service provider, you can use the IoC container to resolve your instance
	$authority = App::make('authority');

## Interface

There are 5 basic functions that you need to be aware of to utilize Authority.

- **allow**: *create* a rule that will *allow* access a resource

	example:
	Authority::allow('read', 'User');

- **deny**: *create* a rule that will *deny* access a resource
	
	example:
	Authority::deny('create', 'User');

- **can**: check if a use *can* access a resource
	
	example:
	Authority::can('read', 'User');

- **cannot**: check if a use *cannot* access a resource

	example:
	Authority::cannot('create', 'User');

- **addAlias**: alias together a group of actions
	
	this example aliases together the CRUD methods under a name of `manage`
	Authority::alias('manage', array('create', 'read', 'update', 'delete'));

## Converting to this library, where you previously had been using the IoC container to resolve an instance.

The service provider will merely create a new instance of Authority, and pass in the currently logged in user to the constructor.  This should be basically the same process that you were doing in your IoC registry.  This means that any code you have used in the past should still work just fine!  However it is recommended that you move your rule definitions into the provided configuration file.