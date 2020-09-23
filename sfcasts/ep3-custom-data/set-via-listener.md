# Setting a Custom Field Via a Listener

When you need to add a custom field... and that custom field requires a service
to populate its value, you have 3 possible solutions: you can create a totally
custom ApiResource class that's not an entity, you can create an output DTO *or*
you can do what we did: add a non-persisted field to your entity. I like this last
option because it's the least... nuclear. If most of your fields come from normal
persisted properties on your entity, creating a custom resource is overkill and
output DTO's - which are *really* cool - come with a few drawbacks.

So that's what we did: we created a non-persisted, normal property on our entity
and populated it in a data provider.

But in reality, there are *multiple* ways that we could set that field. The data
provider solution is the *pure* API Platform solution. The downside is that if
you your `User` object in some code that runs *outside* of an API Plaatform API
call, the `$isMe` field won't be set!

That might be ok - or you might not even have that situation. But let's look
at another idea. What if we create a normal, boring Symfony event listener that's
executed early during the request and we set the `$isMe` field from *there*.

Let's try it! First, remove our current solution: in `UserDataPersister` I'll
comment-out the `$data->setIsMe()` with a common that this will now be set in
a listener. Then over in `UserDataProvider`, I'll do the same thing with the
first `setIsMe()`... and the second.

Sweet! We are back to the broken state where the `$isMe` field is never set.

## Creating the Event Subscriber

To create the event listener, find your terminal and run:

```terminal
php bin/console make:subscriber
```

Well, this will *really* create an event *subscriber*, which I like a bit better.
Let's call it `SetIsMeOnCurrentUserSubscriber`. And for the event, we want
`kernel.request`. Or really, that's its old name. It's *new* name is this
`RequestEvent` class. Copy that and paste it below.

Perfect! Let's go check out the new class in `src/EventSubscriber/`. And...
brilliant! The `onKernelRequest()` will now be called when the `RequestEvent`
is dispatched, which is early in Symfony.

## Populating the Field

So our job is fairly simple: we need to find the authenticated `User` and, if
there *is* an authenticated user, call `setIsMe(true)` on it.

Add a public function `__construct()` with a `Security $security` argument. I'll
hit Alt + Enter and go to "Initialize properties" to create that property and set
it. Then down in `onRequestEvent()`, start with: if not `$event->isMasterRequest()`,
then return.

That's not *super* important, but if your app uses sub-requests, there's no reason
for this code to *also* run for those. If you don't know what I'm talking about
and *want* to, check out our
[Symfony Deep Dive Tutorial](https://symfonycasts.com/screencast/deep-dive/sub-request).

Anyways, get the user with `$user = $this->security->getUser()` and add some
phpdoc above this to help my editor: we know this will be a `User` object or
`null` if the user isn't logged in. If there is *no* user, just return. But
if there *is* a user, call `$user->setIsMe(true)`.

We just set the `$isMe` field on the authenticated `User` object. One cool thing about
Doctrine is that if API platform *later* queries for that *exact* user, Doctrine
will return this *exact* same object in memory, which means that that the `$isMe`
field *will* be properly set.

This means that we will be setting the `$isMe` field for the *current* user, but
purposely *not* setting it for all other `User` objects. So now, in the `User`
class, we should default `$isMe` to `false` to mean:

> Hey! If we did not set this, it must mean that this is *not* the
> currently-authenticated user.

Down in `getIsMe()`, the `LogicException` is no longer needed.

Testing time! At your browser, refresh the item endpoint and... got it! And
if we go to `/api/cheeses.jsonld`... the first item has `isMe: true` and the
others are `isMe: false`.

Next: what if the object that we need to set the field on is *not* the `User`
object? How could we get access to that object inside of the listener? What
if it's a collection endpoint and we need to set a field on *multiple* objects?