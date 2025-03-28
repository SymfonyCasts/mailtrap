# Emails Assertions in Functional Tests

Okay, testing time! If you've explored the codebase a bit, you may have noticed that
someone (it could've been anyone... but probably a Canadian) snuck some tests
into our `tests/Functional/` directory. Do these pass? Idk! Let's find out!

Jump over to your terminal and run:

```terminal
bin/phpunit
```

Uh-oh, 1 failure. Uh-oh, because, truth time, *I'm* the friendly Canadian that added
these and I know they were passing at the beginning of the course! The failure is
in `BookingTest`, specifically, `testCreateBooking`:

> Expected redirect status code but got 500

on line 38 of `BookingTest`. That's where we send the email... so if we're looking
for someone to blame, I feel like we should start with the Canadian, ahem, me and
my wild email-sending ways.

## Foundry and Browser

Open `BookingTest.php`. If you've written functional tests with Symfony before, this
may look a tad different because I'm using some rocking helper libraries. `zenstruck/foundry` gives
us this `ResetDatabase` trait which wipes the database before each test. It also
gives us this `Factories` trait which lets us create database fixtures
in our tests. And `HasBrowser` is from another package - `zenstruck/browser` -
and is essentially a user-friendly wrapper around Symfony's test client.

`testCreateBooking` is the actual test. First, we create a `Trip` in the database with
these known values. Next, some pre-assertions to ensure there are no bookings or
customers in the database. Now, we use `->browser()` to navigate to a trip page,
fill in the booking form, and submit. We then assert that we're redirected to a
specific booking URL and check that the page contains some expected HTML. Finally, we
use Foundry to make some assertions about the data in our database.

## `->throwExceptions()`

Line 38 caused the failure... we're getting a 500 response code when redirecting
to this booking page. 500 status codes in tests can be frustrating because it can
be hard to track down the actual exception. Luckily, Browser allows us to *throw*
the actual exception. At the beginning of this chain, add `->throwExceptions()`:

[[[ code('af097ae44a') ]]]

Back in the terminal, run the tests again:

```terminal-silent
bin/phpunit
```

Now we see an exception: *Unable to find template "@images/mars.png"*. If you recall,
this looks like how we're embedding the trip images into our email. It's failing because
`mars.png` doesn't exist in `public/imgs`. For simplicity, let's adjust our test to use
an existing image. For our fixture here, change `mars` to `iss`, and down below, for
`->visit()`: `/trip/iss`:

[[[ code('477b2e817b') ]]]

Run the tests again!

```terminal-silent
bin/phpunit
```

Passing!

It *looks* like our email is being sent... but let's confirm! At the end of this test,
I want to make some email assertions. Symfony *does* allow this
out of the box, but I like to use a library that puts the fun
back in email functional testing.

## `zenstruck/mailer-test`

At your terminal, run:

```terminal
composer require --dev zenstruck/mailer-test
```

Installed and configured... back in our test, enable it by adding the `InteractsWithMailer`
trait:

[[[ code('f0c9ca029a') ]]]

Start simple, at the end of the test, write `$this->mailer()->assertSentEmailCount(1);`:

[[[ code('2635967f3a') ]]]

## Test-specific Environment Variables

Quick note: `.env.local` - where we put our *real* Mailtrap credentials - is *not*
read or used in the `test` environment: our tests only load `.env` and this
`.env.test` file. And in `.env`, `MAILER_DSN` is set to `null://null`. That's great!
We want our tests to be fast, and not actually sending emails.

Re-run them!

```terminal-silent
bin/phpunit
```

### `assertEmailSentTo()`

Passing - 1 email is being sent! Go back and add another assertion: `->assertEmailSentTo()`.
What email address are we expecting? The one we filled in the form: `bruce@wayne-enterprises.com`.
Copy and paste that. The second argument is the subject: `Booking Confirmation for Visit Mars`:

[[[ code('c404b47d45') ]]]

Run the tests!

```terminal-silent
bin/phpunit
```

Still passing! And notice we have 20 assertions now instead of 19.

### `TestEmail`

But we can go further! Instead of a string for the subject in this assertion, use a closure
with `TestEmail $email` as the argument:

[[[ code('1a0df12e68') ]]]

Inside, we can now make *loads* more assertions
on this email. Since we aren't checking the subject above anymore, add this one first:
`$email->assertSubject('Booking Confirmation for Visit Mars')`:

[[[ code('51bed1a083') ]]]

And we can chain more assertions!

Write `->assert` to see what our editor suggests. Look at them all... Note the `assertTextContains`
and `assertHtmlContains`. You can assert on each of these separately, but, because it's
a best practice for both to contain the important details, use `assertContains()` to check
both at once. Check for `Visit Mars`:

[[[ code('365bc31cb9') ]]]

Links are important to check, so make sure the booking URL is there:
`->assertContains('/booking/'.`. Now, `BookingFactory::first()->getUid()`:

[[[ code('923f543ca7') ]]]

this fetches
the first `Booking` entity in the database (which we know from above there is only the one),
and gets its `uid`.

Heck! We can even check the attachment: `->assertHasFile('Terms of Service.pdf')`:

[[[ code('db553d20bc') ]]]

You can check the content-type and file contents via
extra arguments, but I'm fine just checking that the attachment exists for now.

Go tests go!

```terminal-silent
bin/phpunit
```

Awesome, 25 assertions now!

### `->dd()`

One last thing: if you're ever having trouble figuring out why one of these email
assertions isn't passing, chain a `->dd()`:

[[[ code('1511c955bb') ]]]

and run your tests. When it hits that `dd()`,
it dumps the email to help you debug. Don't forget to remove it when you're done!

Next, I want to add a second email to our app. To avoid duplication and keep things
consistent, we'll create a Twig email layout that both share.
