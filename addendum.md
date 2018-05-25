### [⤺ Back to Table of Contents](/README.md#divergence-framework-documentation)

# Addendum
## About
Much of the code in Divergence owes it's existence to [Chris Alfano](https://github.com/themightychris). The oldest commit I could find was actually made on [May 19, 2012, ActiveRecord](https://github.com/JarvusInnovations/emergence-skeleton/blob/0cccc29c44c5a06c7b0d40d1198fa6219d718fdf/php-classes/ActiveRecord.class.php). At this point the class ActiveRecord was actually being used by us to build dozens of sites and it had been tested in production for years. The framework it belonged to was originally called MICS and you can even see references to it in the comments of the link above. At this point that file was already a few years old. Exactly how old? I have no idea.

2012 was a really different time for PHP. PHP 5.4 and Composer had just come out in March. Docker did not exist. Namespaces were still fresh as they were introduced in 5.3. Chrome was at version 17. A lot of the things we take for granted today simply did not exist or weren't developed enough. Internet Explorer was still at 30% market share. Chrome had just over taken it in May of that year.

However we had a fully working well made and robust ORM and API builder so we went and built hundreds of website. Together with Emergence we had a built in version control system which allowed us to spawn up new sites in seconds at a time when most PHP projects required manual web server and database configurations.

With modern practices much of those versioning techniques became obsolete.

Due to architectural co-dependencies between Emergence (NodeJS) and Emergence-Skeleton (The PHP Code of the Framework) the framework became very difficult to use with third party code which today is essential to the continued success of your code.

I decided to fork many of the classes that come with Emergence and build my own version that is PSR compliant and ready to be packaged to be used with composer.

The work put into unit testing, documenting, and standardizing this framework was done out of pure stubbornness in not wanting to see this elegant collection of code go to waste.

I hope that this code helps you build things with PHP faster.