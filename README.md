# Voters

Useful to check complex user permissions.
###### Inspired from [Symfony voter system](https://symfony.com/doc/current/security/voters.html). 

### Installation

```shell
$ composer require thomas-miceli/voters
```

### Usage

To check a specific permission of an user action `$attribute` (e.g. view or edit) with or without a `$subject` (e.g. an article), every registered voters will be called and will be checked to see if they can vote for the permission.
Every eligible voters will vote whether the user can have the permission or not.

Let's setup permissions for our blog.

#### Create the Permission object

```php
<?php 

$permission = new \Voter\Permission;
$permission->addVoter(new ArticleVoter); // register the voter for this permission object
$user = ...
$article = ...

$permission->can($user, ArticleVoter::VIEW, $article) // check if our user can view the article
$permission->can($user, ArticleVoter::EDIT, $article) // check if our user can edit the article
```

#### Create the voter

```php
<?php

/* This voter determine what the user can do with an article from our blog. */
class ArticleVoter implements Voter
{

    const VIEW = 'view';
    const EDIT = 'edit';

    // if the voter can vote for the requested attribute...
    public function canVote(string $attribute, $subject = null): bool
    {
        return in_array($attribute, [self::EDIT, self::VIEW]) && $subject instanceof Article;
    }

    // ...if yes, the voter will determinate and return the permission for this attribute
    public function vote(?VoterUser $user, string $attribute, $subject = null): bool
    {
        /** @var Article $subject */
        switch ($attribute) {
            // if the user is connected, he can read the article
            case self::VIEW: return $user !== null; 
            
            // if the user is the author, he can modify the article
            case self::EDIT: return $subject->getAuthor() === $user;
        }
        
        // the user has not the permission for other attributes
        return false;
    }
}
```

### Strategies

It's possible to set different strategies to determine the permission when multiple voters are eligible to vote for an attribute.

```php
<?php

$permission = new \Voter\Permission(new \Voter\Strategy\AffirmativeStrategy); // default strategy
// $permission::can() returns true if at least one of the registered voters voted true for the attribute

$permission = new \Voter\Permission(new \Voter\Strategy\VetoStrategy);
// $permission::can() returns true if all the registered voters voted true for the attribute

$permission = new \Voter\Permission(new \Voter\Strategy\MajorityStrategy);
// $permission::can() returns true if at least half plus one of the registered voters voted true for the attribute
```

We can use factory static methods for a better readability.
```php
<?php

$permission = \Voter\Permission::affirmative();
$permission = \Voter\Permission::veto();
$permission = \Voter\Permission::majority();
```

We can create our own strategy and set it to a permission later...

```php
<?php

class CustomStrategy implements \Voter\Strategy\VoterStrategy {
    
    // the permission is granted if at least 4 voters voted true for an attribute
    public function can(?VoterUser $user, array $voters, string $attribute, $subject): bool
    {
        $yes = 0;
        foreach ($voters as $voter) {
            if ($voter->canVote($attribute, $subject)) {
                $vote = $voter->vote($user, $attribute, $subject);
                //ConsoleLogger::debug($voter, $vote, $attribute, $user, $subject);

                if ($vote) {
                    ++$yes;
                }
            }
        }
        return $yes > 3;
    }
}
```

...then use it

```php
$permission = new \Voter\Permission(new CustomStrategy);
```
