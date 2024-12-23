---
title: "Authentication and Standard Webhooks"
author: Phil Curzon
date: Dec 1, 2024
tags: [Haskell, JWT, Authentication, Standard Webhooks]
description: A journey of exploring authentication options for Hereabout and the Haskell implementation of verification of standard webhooks to facilitate user sync between Clerk and Hereabout's Postgres database.
image: haskell-auth.webp
---

>_This is the first in what is planned to be a series of tech blogs about the development of [Hereabout](https://hereabout.uk). Hereabout is a platform to connect people with events happening in their local community. It is currently a proof of concept operating in my local area. Engagement with end users is handled via the Hereabout site while the underlying event and venue metadata is open sourced on community Github repos allowing anyone to contribute to their community as well as build novel applications with the community metadata aggregated there._

When working on a platform that you intend to contain social components, you are very quickly going to have to consider how to support various types of user interaction: comments, likes, reactions, etc. There are lots of interesting questions to answer in the space of social interaction: What sort of content rating systems promote the right user behaviour? How do you encourage thoughtful, high quality content? How do you prevent harmful content or abuse?

Those are all fascinating questions to consider and certainly ones that I'd like to come back to but before you can even think about prototyping anything in this space, you have to consider the elephant in the room and that is the problem of authentication.

You need some mechanism by which users can log in to your site and authenticate against your API so that you can identify who they are and what they're authorised to do.

There are dozens of potential authentication service providers who you can go to to get something off the shelf: Auth0, Clerk, Cognito, Firebase Auth, SuperTokens, etc. Throughout my career, I've worked at many companies that integrated with many of these suppliers but they were always a well established fixture of the technology stack: something that required learning particular APIs or particular support contacts rather than an evaluation of their individual merits and their pricing structures.

### Deciding where to start

Back in 2019, I did some experiments with a Purescript/Halogen UI integrating with [Auth0](https://auth0.com/). Since I had already decided to stick with Purescript/Halogen for the UI of [Hereabout](https://hereabout.uk), this seemed like the obvious place to start. Rather than putting myself to any trouble doing some difficult thinking, I could simply copy and paste some code (with a bit of accounting for the inevitable library version updates since 2019) and have a lovely time. What wasn't there to love?

Well. It turns out that Auth0 and several other authentication providers offer a free plan up to X number of monthly active users (MAUs). This would be absolutely fine if X+1 MAUs cost a nominal fee. Unfortunately, that is not _always_ the case. In the worst possible case, adding 1 MAU might take you from spending $0/month to $1000+/month.

Now, it's possible I might be worrying about nothing here. Perhaps if I have 25,0001 MAUs, it's a sign of success and $1000+/month isn't a big deal but Hereabout is not a VC-backed project, it's a hobby project designed to support local communities, it's at a very early proof of concept stage and I currently don't have any concrete plans to monetise it. Spending a nominal few dollars / month on hosting the backend and postgres database is one thing but worrying about a looming threat of significant monthly fees is not very appealing, especially as a new Dad. So, if for no other reason than my psychological health, cliff-edges in billing are out.

### The open source alternative

My natural reaction to having been burnt by enterprise authentication options was to head straight for the open source alternative. After all, why risk paying for anything but the cost of hosting the authentication platform?  So, I started investigating [Keycloak](https://www.keycloak.org/).

I enthusiastically fired up `docker compose` to get an instance running and start figuring out how to make use of it.

Now, Keycloak is fabulous, it has every feature under the sun and you don't have to pay for any of it you just have to work through the various modules and rigorously set them up.

I managed to make some progress with my local keycloak configuration but realised there were a lot of subtleties, particularly differences in configuration required for running Keycloak in a development setting vs a production setting. Obviously, with something as sensitive as an auth service, this can create some pretty nasty security vulnerabilities if not done correctly.

After sinking a couple of days into Keycloak I realised I wasn't making enough progress. My objective here was never to learn the ins and outs of configuring an authentication provider myself, it was to get a working login system so that I could develop other features I actually cared about.  So, as fascinating as it was, I decided Keycloak was out.

### What now?

Back I went to the research stage. I started seriously looking into [Supertokens](https://supertokens.com/). On their pricing page, they make a big selling point of the fact that there is no 10x increase in price when you cross some cliff-edge in user count. 

It's worth noting that Supertokens actually has a less generous free tier than Auth0 but you pay per user above the free tier threshold rather than paying per user on all of your users once you've reached the free tier limit. That means you're never in the position of one new user costing you thousands.

There is also a self-hosted open source option which would work out to be very reasonably priced even if Hereabout were to end up being quite successful.

Unfortunately, Supertokens offers backend integrations with NodeJS, Go and Python. Not at all helpful for my Haskell API. I'm sure it's possible to get Supertokens working with my Haskell backend by spinning up an extra service in one of those languages but my appetite for that much complexity is very low so, despite the appeal of the self-hosted offering, Supertokens was also out.

### A solution at last

Finally, and with much suspicion by this point, I landed on [Clerk](https://clerk.com/). Clerk also doesn't have a cliff-edge in billing and it includes fairly straightforward instructions for integrating with generic JWT libraries on the backend and it has a raw javascript library on the client-side that I can readily wrap with Purescript.

Helpfully, Clerk also comes with a bunch of pure javascript components for account management, user login and such that can be rolled into a Purescript/Halogen UI very easily and that's another convenient time saver when it comes to getting something up and running quickly.

So, having now tackled the first problem I started worrying about another. If I'm outsourcing user login to a third party, how on earth do I get easy access to user data so that I can facilitate building the social features that I actually care about?

### The webhook reality check

When working on the auth stack at a well established company, there are all kinds of options to get user data: user microservices, caches, kafka, etc. These are all pretty high in complexity budget and I want to keep my stack as simple as possible for as long as possible.

The easiest option is probably just polling for user info when required but that introduces extra latency at every stage of my app. I also have to consider there are probably going to be rate limits on the auth service API (I checked: there are) that I'm going to fall foul of at only a very tiny number of regularly active users.

In the end, I decided to use Clerk's webhooks so that user information would be constantly sent to my Haskell API and update my postgres database with the latest information. This would obviously be particularly lovely attached to Kafka, publishing some user updated topic to my app but I simply don't need that at this stage.

It turns out that Clerk uses [Svix](https://svix.com/) to send its webhooks. Svix have in turn created a standard called [Standard Webhooks](https://www.standardwebhooks.com/).

Basically, there is a digital signing process to these webhooks so that we can ensure that the webhooks are coming from a trusted source.   Libraries are provided for the implementation of the digital signing process but, naturally, there is nothing for Haskell. Before tearing my hair out and going back to square one again, I realise there is a [guide](https://docs.svix.com/receiving/verifying-payloads/how-manual) for verifying the payloads manually.

They send send three HTTP headers `svix-id`, `svix-timestamp` and `svix-signature`.  You can reproduce `svix-signature` with the `SHA256 HMAC` of `${svix_id}.${svix_timestamp}.${http_request_body}` and your shared `HMAC` secret.

### The Haskell implementation

At this point, the job is just to reproduce the example javascript in Haskell. Thankfully, this is reasonably straightforward. There is a [simulation tool](https://www.standardwebhooks.com/simulate) to check your payloads which can be easily translated into tests.

Just watch out for the fact that the secrets are provided in the form `whsec_[SECRET]`, e.g. `whsec_FNjuUR17qqxt6GtORAHn6kLa`. You need to chop out the `whsec_` prefix and `base64` decode the suffix before feeding it into the function below.

```haskell
{-# LANGUAGE OverloadedStrings #-}

module Hereabout.Webhooks.Standard where

import Data.ByteString (ByteString)
import Crypto.Hash (Digest)
import Crypto.Hash.Algorithms (SHA256)
import qualified Data.ByteArray as Mem
import Data.Fixed (Pico)
import qualified Data.Text as T
import qualified Data.Text.Encoding as TE
import Control.Monad.Reader (reader)
import Crypto.MAC.HMAC (HMAC(..), hmac)
import qualified Data.ByteString.Base64 as Base64
import qualified Data.Time.Clock.POSIX as Time
import qualified Data.Text.Read as TR
import qualified Data.Time as Time
import Hereabout.Effects

data StandardWebhookError
  = StandardWebhookTimestampExpired
  | StandardWebhookSignatureError
  | StandardWebhookTimestampMalformed
  deriving (Eq, Ord, Show)

data StandardWebhookVerifiedData = StandardWebhookVerifiedData 
  { swhVerifiedData :: ByteString
  , swhId :: T.Text
  }
  deriving (Eq, Ord, Show)

verifyStandardWebhook :: (CurrentTime m) => ByteString -> T.Text -> T.Text -> T.Text -> ByteString -> m (Either StandardWebhookError StandardWebhookVerifiedData)
verifyStandardWebhook hmacSig swhId' swhTimestamp swhSignature rawData = do
  let sigs = last . T.split (== ',') <$> T.split (== ' ') swhSignature
  let signedData = TE.encodeUtf8 swhId' <> "." <> TE.encodeUtf8 swhTimestamp <> "." <> rawData
  let (reconstructSignature :: ByteString) = Mem.convert (hmacGetDigest $ hmac hmacSig signedData :: Digest SHA256)
  let base64Sig = TE.decodeUtf8 $ Base64.encode reconstructSignature
  posixTime <- currentPOSIXTime
  case (\(i, _) -> fromIntegral i) <$> TR.decimal swhTimestamp of
    Right (swhTime :: Pico) -> do
      let timeDiff = abs (Time.nominalDiffTimeToSeconds posixTime - swhTime)
      if elem base64Sig sigs && timeDiff <= 10 then do
        pure . Right $ StandardWebhookVerifiedData { swhVerifiedData = rawData, swhId = swhId' }
      else if elem base64Sig sigs then
        pure $ Left StandardWebhookTimestampExpired
      else
        pure $ Left StandardWebhookSignatureError
    Left _ -> pure $ Left StandardWebhookTimestampMalformed
```

The `CurrentTime` typeclass just facilitates unit testing with hard-coded values. You can implement it:

```haskell
class Monad m => CurrentTime m where
  currentPOSIXTime :: m POSIXTime

instance CurrentTime IO where
  currentPOSIXTime = getPOSIXTime
```

And then unit tests look something like this:

```haskell
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE QuasiQuotes #-}
module Server.StandardWebhookTests where

import Test.Tasty
import Test.Tasty.HUnit
import qualified Data.Text as T
import Data.ByteString (ByteString)
import Hereabout.Effects
import qualified Data.ByteString.Base64 as Base64
import qualified Data.Text.Encoding as TE
import Text.RawString.QQ(r)
import Data.Time.Clock.POSIX (POSIXTime)
import Control.Monad.Reader (ReaderT (runReaderT), MonadReader (ask))
import Hereabout.Webhooks.Standard (StandardWebhookVerifiedData(..), StandardWebhookError (..), verifyStandardWebhook)

newtype FixedTS a = FixedTS (ReaderT POSIXTime IO a) deriving (Functor, Applicative, Monad, MonadReader POSIXTime)

unFixedTS :: POSIXTime -> FixedTS a -> IO a
unFixedTS time (FixedTS stuff) = runReaderT  stuff time

instance CurrentTime FixedTS where
  currentPOSIXTime = ask

swhTests :: TestTree
swhTests = testGroup "Standard Webhook tests"
  [ testCase "Webhook verification succeeds with correct signature and matching timestamp" $ do
    result <- unFixedTS 1732892252 $ verifyStandardWebhook secret msgId timestamp sig data'

    case result of
      Right data'' -> swhVerifiedData data'' @?= data'
      Left err -> fail $ show err

  , testCase "Webhook verification succeeds with correct signature and timestamp within (10s) tolerance" $ do

    result <- unFixedTS 1732892262 $ verifyStandardWebhook secret msgId timestamp sig data'

    case result of
      Right data'' -> swhVerifiedData data'' @?= data'
      Left err -> fail $ show err

  , testCase "Webhook verification fails with correct signature and when outside tolerance (11s)" $ do

    result <- unFixedTS 1732892263 $ verifyStandardWebhook secret msgId timestamp sig data'

    case result of
      Right _ -> fail "Expected failure"
      Left err -> err @?= StandardWebhookTimestampExpired

  , testCase "Webhook verification fails with wrong signature with correct timestamp" $ do

    result <- unFixedTS 1732892252 $ verifyStandardWebhook secret msgId timestamp wrongSig data'

    case result of
      Right _ -> fail "Expected failure"
      Left err -> err @?= StandardWebhookSignatureError

  , testCase "Webhook verification fails when timestamp malformed" $ do

    result <- unFixedTS 1732892252 $ verifyStandardWebhook secret msgId malformedTimestamp wrongSig data'

    case result of
      Right _ -> fail "Expected failure"
      Left err -> err @?= StandardWebhookTimestampMalformed
  ]
  where 
    data' :: ByteString = TE.encodeUtf8 [r|{"event_type":"ping","data":{"success":true}}|]
    msgId :: T.Text = "msg_8Js4eVSDtRjxfQvP"
    timestamp :: T.Text = "1732892252"
    malformedTimestamp :: T.Text = "James T. Kirk"
    sig :: T.Text  = "v1,WCViVA6U2SyPxf8BXEeRiGIJvtOJeJG3nrUTv2w89Kc="
    wrongSig :: T.Text  = "v1,WCViVA6U2SyPxf8BXEeRiGIJvtOJeJG3nrUTv2w89Bc="
    secret :: ByteString = Base64.decodeLenient $ TE.encodeUtf8 "8tVXGUy1IpTHVstt9AS5VZL4"
    secret :: ByteString = Base64.decodeLenient $ TE.encodeUtf8 "8tVXGUy1IpTHVstt9AS5VZL4"
```

### Concluding thoughts

None of this was an entirely smooth experience: the pricing model of authentication providers is both complex and intimidating to smaller projects. Minimal or non-existent Haskell library support and lack of language agnostic documentation for even some of the more popular providers makes getting started a bit of minefield.

Auth providers like Supertokens that have a very small set of supported languages (they don't even provide SDKs for Java or .NET) might do well to put a lot of effort into documentation so that the community can provide such SDKs quickly and easily, backed by solid reference material.

In the future, I might go back and release a library on Hackage that provides the above webhook support with a more polished API but, until then, you can consider the above code available under standard 3-Clause BSD licence terms in case it's helpful to anyone building a Haskell project that requires syncing data from Clerk.
