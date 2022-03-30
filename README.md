# How to use Postgres Views and Stored Procedures with Catalyst


The question this essay answers is how to connect a Catalyst application to a Postgres back-end using stored procedures and views.  I’m not trying to convince anyone this is the correct way, I’m providing a guide for those who have already decided that.  If you’re trying to decide how to approach connecting your Catalyst application to Postgres, there are better resources available.

There are a number of design considerations that went into the example code and have nothing really to do with the question at hand.  The only one I’ll mention is that I consider the data model built in the Catalyst application as part of the codebase not the database.  It shouldn't matter what gyrations I put the table structure through, I shouldn't have to touch the code-base to deal with it.  If the queries are built into the code, as is common, most database changes will force a code change.  This is not really desirable.  The application, using the data model as a translation layer, should always know what to expect and the database should always know what to provide; each is left to act independantly to accomplish what is needed.  This is very helpful in larger projects where the guys writing the application are not the guys dealing with the DB.  It would also be helpful in any group project where the ability to work on the DB separately from the application is appropriate.

In order for the code included to work, you’ll need a few things installed and completed.  First you need Catalyst along with Moose and the Postgres drivers.  This essay assumes you already have Catalyst installed and working, Postgres installed and working and have successfully made them talk to each other.  I also assume at least moderate comfort working with Catalyst, Postgres and SQL.  You won’t need to be a senior developer to follow along and understand, but I’m likely to leave out details I see as universal or standard, counting on your knowledge to fill in those gaps.

Catalyst applications with Postgres in the back-end will commmonly use the create script to autogenerate the schema used by catalyst:

`/myapp_create.pl model MainDB DBIC::Schema myapp::Schema create=static components=TimeStamp 'dbi:Pg:host=mydb.domain.net dbname=mydb' 'skippy' 'password' '{ AutoCommit => 1 }'`

Your exact command will probably be slighltly different.  I assume something similar to this is what is being used if you follow along.  The first thing you need to know is that autodiscovery of the schema doesn’t include stored procedures.  That means, when we get there, you will have to manually add the required schema elements.  It’s probably important to note that the autogenerated schema names will differ from the actual object names.  You actually get the tables as well as the views, but in the approach here, they can be safely ignored.

We can start by looking at our only table, it should seem pretty straight forward.  It's one table out of a much larger system and it’s been slightly modified for the example at hand:

```
CREATE TABLE a01_user (
  userid integer NOT NULL DEFAULT nextval('a01_user_userid_seq'::regclass),
  username character varying NOT NULL,
  fname character varying,
  lname character varying,
  accountid integer NOT NULL,
  password character varying NOT NULL,
  email character varying,
  administrator boolean NOT NULL DEFAULT false,
  disabled date,
  CONSTRAINT a01_user_pk PRIMARY KEY (userid)
) WITH ( OIDS=FALSE);
ALTER TABLE a01_user OWNER TO skippy;
```

I demonstrate a few bad habits with this table.  First, the use of a synthetic key (funny numbers as my DB mentor used to say), although convenient has some drawbacks and ignores the probably useable username.  This table also intentionally packs more in than is strictly correct with a normalised data model. 

I’m sure there’s a question about the table name.  I like to be able to sort objects by type and this helps.  For example, a1_, a2_, etc. might represent key tables, d1_, d2_, etc. might be look-up/detail tables, v1_, v2_, etc. are views... you get the idea.  When you’re looking at a data model with a couple dozen tables, or worse the SQL that created it, putting a meta-class identifier in the object name can save a lot of time and pain.

Next, given that we don’t want our code to directly interact with our tables, we’ll need some DB-side interface pieces, which are the views and stored procedures.   So, first, the views:

```
CREATE OR REPLACE VIEW v01_users AS 
 SELECT a01_user.userid,
    a01_user.accountid,
    a01_user.email,
    a01_user.fname,
    a01_user.lname,
    a01_user.username,
    a01_user.administrator
   FROM a01_user WHERE disabled IS NULL;
ALTER TABLE v01_users OWNER TO skippy;

CREATE OR REPLACE VIEW sv01_credential AS 
 SELECT a01_user.userid,
    a01_user.username,
    a01_user.password
   FROM a01_user
   WHERE disabled IS NULL;
ALTER TABLE sv01_credential OWNER TO skippy;
```

The first thing to notice is that each view provides a different data set starting from the same table.  It should be clear that getting user data should not include the password (even hashed, of course) and vice versa.  Each also represents one specific way of looking at (a view) of the thing in question: a user record and a credential.

Correspondingly, inside the catalyst application you’ll need a few things.  First, is a function that needs to get at the data (for example to validate a credential, this example uses the PBKDF2 library)):

```
sub connect  {
	my ($self, $c) = @_;
	my $pbkdf2 = Crypt::PBKDF2->new(HASH_CLASS 	=> 'HMACSHA2',
					salt_len 	=> 10,
					output_len	=> 50,
					);
	if( my $UserName = $c->request->body_parameters->{username}
		and my $attempt = $c->request->parameters->{password} )  {
			my $Credential = $c->model('MainDB')->get_key($UserName);
			if ($pbkdf2->validate($Credential, $attempt))  {			
				my $UserBlock = $c->model('MainDB')->get_user($UserName);
				//Write user information to session object
			} else  {
				//Clear any existing session object, create empty one
			}
	}
	$c->response->redirect($c->uri_for('/'));
	return;
}
```

First, this function fetches the key from the application data model, `$c->model('MainDB')->get_key($UserName)`.  The function in the model is quite short:

```
sub get_key  {
	my ($self, $UserID) = @_;
	my $RecordSet = $self->resultset('Sv01Credential')->search({userid=>$UserID});
	my $RecordBuffer = $RecordSet->next;
	my $userkey = $RecordBuffer->password;
	return $userkey;
}
```

The provided credential is validated against the stored one and if they match (according to pbkdf2) we then grab the user data with 
`my $UserBlock = $c->model('MainDB')->get_user($UserID);`

The corresponding model piece for this call:

```
sub get_user  {
	my ($self, $UserID) = @_;
	my $RecordSet = $self->resultset('V01User')->search({userid=>$UserID});
	my @RecordRow = $RecordSet->all;
	return \@RecordRow;
}
```

Again, really short and sweet.

So now that we have the retrieve portion built, what have we accomplished?  Well, we've demonstrated the data as seen by the application is independant of the table structure.  What’s nice is that as long as the views are consistent with what they return, you can change the table structure to your heart’s content and the application will always work without additional effort.  Want it to be 3 tables?  Go for it, just make sure the view's result set is consistent.  Also, you’re now relying on the database to filter on details like whether or not an account is expired (it’s in the definition, really, go look).  Most importantly, you can add a uniqueness constraint on username for example (in the code this was ripped from, the key was compound and I didn't feel like dealing with that, hence the funny number key) and not have to worry about handling this in your code.  

An added benefit is input sanitization.  When using a view or stored procedure, everything is paramaterised.  You’re not writing the SQL in code (which is a very bad practice), you’re using proper, validated libraries to autogenerate the SQL based on your need and mapping to the provided API.  This guarantees you can never be susceptible to injection attacks, since the DB engine will always interpret the parameter as text, never as executable code.
Everything up to this point will just work with Catalyst and Postgres.  Getting stored procedures working takes a bit more effort (special thanks to the Perl Monks, wihout whose wisdom I would never have figured this out).  First thing you need is a stored procedure:

```
CREATE OR REPLACE FUNCTION p02_updateaccount(
    uname text,
    firstname text,
    lastname text,
    emailaddr text,
    pword text,
    administ boolean,
    dlock boolean)
  RETURNS integer AS
$BODY$
DECLARE
	accountcheck integer;
	pwcheck text;
BEGIN
	select userid into accountcheck from a01_user where username = uname;
	/*
		This isn’t about writing Postgres stored procedures, so most of this is removed.  
		There is lots of help for writing these things just a Google search away.
	*/
	 select lastval() into accountcheck;
	return accountcheck;
END;
$BODY$
  LANGUAGE plpgsql VOLATILE
  COST 100;
ALTER FUNCTION p02_updateaccount(text, text, text, text, text, boolean, boolean) OWNER TO skippy;
```

The important points to note are the function definition and the return.  In this case, the return is simply the userid/primary key I just updated or added.  The parameters to the function will be required when we put together the call in the Catalyst application.

The next piece is to add the stored procedure to the schema that catalyst works from.  With tables and views, this works automatically using the create script as mentioned at the top.  Create a file with an appropriate name (Update_account.pm) and put it in your Schema/Result directory.  Then, you need the content:

```
use utf8;
package myapp::Schema::Result::Update_account;
=head1 NAME
myapp::Schema::Result::Update_account
=cut
use strict;
use warnings;
use Moose;
use MooseX::NonMoose;
use MooseX::MarkAsMethods autoclean => 1;
extends 'DBIx::Class::Core';

__PACKAGE__->table("NONE");
__PACKAGE__->add_columns(qw/retval/);
__PACKAGE__->result_source_instance  ->name(\'(select p02_updateaccount(?::text, ?::text, ?::text, ?::text, ?::text, ?::BOOLEAN, ?::BOOLEAN) as retval)');
1;
```

The important point is that the call definition with the placeholders has to exactly match the function call as given in Postgres or it will throw errors.  It’s parameterised as mentioned, so safer than using strings.  The return value is captured in retval and will be available as from any query.
You also need the piece in the model file that accesses it:

```
sub update_account  {
	my ($self, $username, $firstname, $lastname, $email, $password, $admin, $dlock) = @_;
	my $ReturnRef = {};
	//Snipped out
	my($returnval) = $self->resultset('Update_account')->search({}, {bind => [$username, $firstname, $lastname, $email, $HashedVal, $admin, $dlock]});
	my $retval = $returnval->retval;
	//Again, snip, fill in as you require
}
```

The line that does all the work, `my($returnval) = $self->resultset('Update_account')->search({}, {bind => [$username, $firstname, $lastname, $email, $HashedVal, $admin, $dlock]});`, simply binds the variables to the SQL call then sends it off to Postgres putting the returned value in $returnval, which is a hash ref.

I’m leaving out the controller piece, I feel it shouldn’t be necessary as it would mostly be about a call to the model with the correct parameters.  There are a couple examples of what that looks like already.

It might seem there are a lot of moving parts here, but really it’s not different from what you would usually be doing with Catalyst, just sometimes you lose the syntactic sugar coating we all love.  Simply explained:

1. Create tables, views and stored procedures, etc. appropriate to your needs and current database,
2. Add the tables and views to the schema using the create script,
3. Add the stored procedures to the schema by adding the files to the Schema/Result directory,
4. Add connectors to the model,
5. Use as required in your controller.
6. 
Enjoy!  
