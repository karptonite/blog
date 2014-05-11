---
layout: post
title: "Rehashing Password Hashes"
---

Every time a prominent website has their user database stolen, the question of
how the passwords were stored comes up. Even properly hashed password tables
are vulnerable to some attacks, but the better the hashing algorithm, the less
the risk to your users.

Tools have come a long way. In the PHP world, we now have the `password_hash()`
function [built into PHP
5.5](http://docs.php.net/manual/en/function.password-hash.php). However, many
of us work on websites which have a table of old passwords hashed by older,
less secure tools or by some home-brewed hashing system. When I read about
methods of upgrading old systems, the recommendation I sometimes see is that
the best you can do to correct this situation is have a system that validates
passwords when users log in using the old system, then [silently
rehashes](https://github.com/jeremykendall/password-validator#upgrading-legacy-passwords)
them with the new improved algorithm.  Unfortunately, this leaves infrequent
users, or users who created accounts and never logged in again, more
vulnerable, should your database be compromised.  But you can't create the
correct new hash without the original password,
[right](http://blog.stidges.com/post/upgrading-legacy-passwords-with-laravel)?

This is absolutely correct;
luckily, there is another alternative: you can rehash the old _hashes_.
Here is how it works:

Say your legacy systems works roughly as follows:

{% highlight php startinline %}
public function savePassword($user, $password)
{
   $salt = $this->getRandomSalt();
   $hash = $this->hashPassword($password, $salt);
   $user->setPasswordHash($hash);
   $user->setPasswordSalt($salt);

public function checkUserPassword($user, $password)
{
   return $this->checkPassword($password, $user->getSalt(), $user->getHash())
}

public function checkPassword($password, $salt, $hash)
{
   return $hash == $this->hashPassword($password, $salt);
}
{% endhighlight %}

When you want to upgrade to password_hash, run something resembling the following on
each user.
{% highlight php startinline %}
public function convertUserPassword($user)
{
   $user->setPasswordVersion('legacy');
   $user->setNewPasswordHash(password_hash($user->getHash()));
}
{% endhighlight %}

Once the passwords are converted, you can change your password checking code:

{% highlight php startinline %}
public function checkUserPassword($user, $password)
{
   if($user->getPasswordVersion() =='legacy')
   {
      // using our old hashPassword function and our old salt
      $oldStyleHash = $this->hashPassword($password, $user->getSalt());
      return password_verify($oldStyleHash, $user->getNewPasswordHash());
      // if you want, now might be a good time to hash the actual password,
      // and upgrade the user's password version.
   }
   // else, use password_verify() as normal
}
{% endhighlight %}
Notice what we've done here: We don't have the user's password, but we do have a hash
that we know can be generated from the password and the proper salt. We've treated that
hash as the _password_ for the new, improved hashing system. We know that when the user
does type in their password, we will be able to regenerate that hash because we
still have the salt.

Now for the important part: you can delete the old password hashes (but not the
salts) from the database. All users are now protected by the new password
hashing algorithm, even if they never log in again.

Note that in practice, some legacy systems will have saved the salt and the hash
as part of the same string, but these should be separable. The important thing is to keep the salt
but discard the old hash.

Once the original passwords are deleted, ALL of your users should benefit from the improved hashing
algorithm, not just those who log in again.

Note: I am by no means a security expert. If I'm wrong about something in this article, please tell me!
