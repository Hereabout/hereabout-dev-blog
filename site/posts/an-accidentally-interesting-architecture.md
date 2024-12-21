---
title: "An Accidentally Interesting Architecture" 
author: Phil Curzon
date: Dec 2, 2024
tags: []
description: The circuitous development of the architecture of Hereabout to facilitate community contributions which ultimately results in an accidentally interesting architecture resembling CQRS.
---

Many important discoveries in human history have been made thanks to fortunate happenstance. Alexander Fleming returned from a holiday to find a mouldy petri dish in his lab that was suspiciously free of bacteria which led to the discover of penicillin. Walter Jaegar discovered that his poison gas detector was triggered by his cigarette, a discovery that resulted in the invention of the smoke detector.

My architectural discovery was hardly world-shattering but it was nonetheless _interesting_.

**The problem**

I have wanted to build Hereabout for quite some time. Conceptually, it is very simple. Hereabout captures `Events` by location, by time (accounting for repeating events) and stores titles, descriptions, images, etc. It also captures `Venues` that those events might be hosted at. Then, by throwing in some mapping data, it presents the user with a hyperlocal feed of events and venues in their `Community`.

I built a front end to display all this before really thinking about the sticking point: how on earth are users actually going to add their events and venues to the site?

I experimented with various options for building a UI to accommodate this. I tried various markdown editors to allow people to edit their event descriptions, I investigated image upload mechanisms, I looked into assigning user roles so that certain users could be associated with particular venues. There are a lot of challenges:

- How do you deal with bad actors?
  - How do you verify users who say they are associated with a particular venue actually are?
  - Can they break the layout of your page?
  - Can they upload malicious scripts and get your site to serve them?
  - How do you avoid harmful images being uploaded and what do you do when they inevitably are?
- How do you moderate the content?
  - How do users report content as problematic?
  - How do you roll back problematic revisions?
- How do you make the experience of working with the platform not horrible with limited time and resources?

None of these problems are insurmountable but they are far easier when you're an established company with dedicated teams to take on some of this work.

**Rolling the dice**

To unwind, I like role-playing games. I like them both as a player and as a game master, designing campaign worlds and rules systems. Last year I started working on a role-playing system designed to solve a couple of issues that I didn't like about existing systems. I also wanted to be able to support many different campaign worlds and feels, from sci-fi settings to medieval fantasy. My plan was to design a core ruleset and then make the system extensible.

Writing rules system for games is quite a tedious process, it involves dealing with large quantities of text with many common snippets having to be inserted liberally throughout the text. If you partially re-design the system part way through as I continue to do as I get new ideas, you are going to end up doing a lot of painful manual editing and you still risk leaving inconsistencies. I'd been there before and I didn't fancy it again.

Fortunately, being a software professional, there was an obvious answer: just write the rules as code and then print the rules into a nice readable format. So, that's what I did. I wrote the core of my RPG in Haskell and then put the configuration in Dhall.  _Skills_, _Weapons_, _Armour_, etc. are all represented in Dhall.

In principle, that means I could distribute a set of core components containing building blocks that should be useful in most RPGs regardless of setting while other people could create repositories with custom components for their own themes or their own games. The end user, presumably a game master, can then aggregate the set of components they want for their game and run the site generator to generate a themed rules website with exactly the bits they wanted to include and nothing else.

**Higher Kinded Data (HKD)**

Recently I was discussing Higher Kinded Data with one of my former colleagues who authored [higgledy](https://hackage.haskell.org/package/higgledy). Higher Kinded Data is an interesting pattern because it lets you reuse the the same structure to represent partial and complete data. If we consider an event:

```haskell
data Event = Event
  { eventName :: Text
  , eventTimestamp :: UTCTime
  , eventPlace :: Place
  ...
  }
```

This becomes: 

```haskell
data Event f = Event
  { eventName :: f Text
  , eventTimestamp :: f UTCTime
  , eventPlace :: f Place
  ...
  }
```

where `f` might be `Identity`, `Maybe`, etc.

This is very handy if you are doing CRUD operations on some type. You can `Create` with the complete version parametrised with `Identity` and you can `Update` with the version parametrised with `Maybe` to indicate which fields should be updated and which are unchanged.

You can come up with something like this:

```haskell
class HasUniqueId a where
  type UniqueIdOf a
  uniqueId :: a -> UniqueIdOf a

class (Monad m, HasUniqueId (a Identity)) => CRUD m a where
  create :: a Identity -> m ()
  delete :: Proxy a -> UniqueIdOf (a Identity) -> m ()
  read :: UniqueIdOf (a Identity) -> m (Maybe (a Identity))
  update :: UniqueIdOf (a Identity) -> a Maybe -> m ()
```

- `create` requires fully populated data (parametrised with `Identity`) so that we insert a complete record.
- `delete` requires a proxy for instance resolution and just the `id` we want to delete.
- `read` requires just requires the `id` we'd like to try and get.
- `update` requires the `id` we'd like to update and partially populated data (parametrised with `Maybe`).

The actual implementation in Hereabout is a bit more complicated than this because it also has to deal with lists. Those lists may include elements that individually require a create, delete or update.

**The beginnings of an interesting architecture**

It was about at this point that I realised what should have been staring me in the face from the beginning. There was already a very well established system that had all of the attributes I needed for community management in Hereabout. It was a something I already used every day: Github.

- It has pull requests in which the merits of proposed changes can be discussed.
- It has code reviews to, amongst other things, prevent harmful content from being merged to main.
- It has permissions and controls to require reviews by particular owners at a granular level.
- It has branch protection to prevent changes going into `main` until they have been reviewed.
- It has actions that can be run on each commit.
- It allows you to easily rollback unwanted changes.

So, with Infrastructure as Code now being well established. It was time for the dawn of the Pub Quiz as Code.

By combining the ideas of my role-playing system and my events platform. I had a solution to my problems: I would represent the `event` and `venue` metadata in a public Github repo allowing anyone to contribute to it.

I would build a CLI that would allow me to diff the Dhall configuration in the github repo against that stored in Hereabout's postgres database. Diffs could be represented using Higher Kinded Data and rendered back into Dhall or transformed into CRUD operations against the database - the idea was to create a feel similar to `terraform plan` and `terraform apply`.

Effectively, this means Hereabout operates somewhat like a Wiki but instead of collaboratively edited _text_, Hereabout is based on collaboratively edited _metadata_.

Pull Requests on Github could run `plan` allowing people to preview the changes that would be made. Once merged, `apply` would be run against commits to `main` to make those changes live.

Obviously using Dhall and Github is not terribly end-user friendly. I'm not expecting pub landlords to start firing up `nix` to get themselves a working `Dhall language server` so they can open a pull request to add their Tuesday night quiz. Rather, for now, the important thing is that some kind of community-facing tooling exists. I can probably figure out how to build a friendlier update flow if Hereabout gains sufficient traction to warrant it.

> Note: Code review doesn't represent a fool-proof solution for stripping out potentially harmful scripts or raw html so Hereabout also uses Pandoc to strip out problematic markdown. I might talk about this in a future entry if there's anything sufficiently interesting to write about.

**The unintended good**

When it came to actually setting up a production deployment of Hereabout a few weeks back, I realised the above configuration had brought even more benefits than I'd expected. Considering the website is essentially read-only when it comes to events and venues, its database access can be heavily locked down in terms of permissions, significantly reducing the risk of the site being compromised.

This is not particularly novel, it's essentially just Command Query Responsibility Segregation (CQRS). This kind of pattern is actually particularly well suited to the Hereabout use case: unlike reads, writes involve far more complicated geospatial queries to generate foreign keys into the relevant `Communities`. Having them completely segregated in an entirely asynchronous Github actions pipeline should help keep the site snappy and responsive around this.

The CLI also benefits from this in that each repo is designed to cover a particular area of the country so there is a well defined `Area` which it is allowed to apply updates to. These areas are aligned with _local authority areas_ and you can find the example for Elmbridge [here](https://github.com/Hereabout/area-uk-elmbridge).

