# Cards Library for Bot Builder v4 .NET SDK

## Build status
| Branch | Status | Recommended NuGet package version |
| ------ | ------ | ------ |
| master | [![Build status](https://ci.appveyor.com/api/projects/status/b9123gl3kih8x9cb?svg=true)](https://ci.appveyor.com/project/garypretty/botbuilder-community) | [Available via NuGet](https://www.nuget.org/packages/Bot.Builder.Community.Cards/) |

## Description

This is part of the [Bot Builder Community Extensions](https://github.com/botbuildercommunity) project which contains various pieces of middleware, recognizers and other components for use with the Bot Builder .NET SDK v4.

The cards library currently has three main features:

1. Both Adaptive Cards and Bot Framework cards can be disabled
2. Adaptive Cards can have their input values preserved when they are submitted in Microsoft Teams
3. Adaptive Cards can be translated

More information about the cards library can be found in its original ideation thread: https://github.com/BotBuilderCommunity/botbuilder-community-dotnet/issues/137

## Installation

Available via NuGet package [Bot.Builder.Community.Cards](https://www.nuget.org/packages/Bot.Builder.Community.Cards/).

Install into your project use the following command in the package manager. 

```
    PM> Install-Package Bot.Builder.Community.Cards
```

## Sample

A sample bot showcasing the various features of the cards library is available [here](https://github.com/BotBuilderCommunity/botbuilder-community-dotnet/samples/Cards%20Library%20Sample).

## Usage

### Disabling and deleting

The cards library can automatically decide whether to disable or delete a card based on the capabilities of the channel, much like how `ChoiceFactory` automatically decides how to display choices in the Bot Builder SDK. If a channel supports message updates then the cards library will try to delete the card because that's assumed to be the preferred user experience, whereas if the channel does not support message updates then the card will simply be disabled using bot state which works on every channel.

The classes used for disabling and deleting cards are in the `Bot.Builder.Community.Cards.Management` namespace, because the functionality makes use of a *card manager*. The card manager uses *card manager state* to track ID's and save activities.

Tracking ID's is how the cards library effectively disables cards even when the channel does not support message updates. There are two *tracking styles*: `TrackEnabled` and `TrackDisabled`. When the tracking style is `TrackEnabled` then the cards library will treat any ID tracked in bot state as enabled and any ID not tracked in bot state as disabled. Likewise, when the tracking style is `TrackDisabled` then the cards library will treat any ID tracked in bot state as disabled and any ID not tracked in bot state as enabled.

In order for ID-tracking to work, each ID needs to be present in a card's *action data*. The cards library defines action data as an object found both in an outgoing card attachment sent from the bot to a channel and in an incoming activity sent from the channel to the bot when a user interacts with the card. In an Adaptive Card, action data is what's in a submit action's `data` property. In a Bot Framework card action you can usually expect action data to be in the `value` property and you can likewise usually expect action data to be in the `value` property of an incoming activity. We have to say "usually" because some channels behave a little differently but the cards library is able to automatically adapt to the behavior of those channels.

The cards library can automatically put ID's in action data and it calls these ID's *action data ID's* or just *data ID's* for short. A data ID found in an incoming activity can be compared to the ID's tracked in card manager state to see whether the ID is enabled or not. If the ID is disabled then the bot can ignore the incoming activity, making it look as though nothing happened when the card was clicked. That is the "disabled" effect the cards library provides.

There are four kinds of data ID's to account for the different *data ID scopes*. The scopes refer to how much is getting disabled or deleted.

1. An *action ID* will only disable or delete one action, whether it's a Bot Framework card action or an Adaptive Card submit action. This means the ID is scoped to an action, i.e. it has the action scope.
2. A *card ID* will disable or delete an entire card which may contain multiple actions. This means the ID is scoped to a card, i.e. it has the card scope.
3. A *carousel ID* will disable or delete an entire carousel which may contain multiple cards. This means the ID is scoped to a carousel, i.e. it has the carousel scope.
4. A *batch ID* will disable or delete an entire batch of activities which may contain multiple carousels (i.e. multiple activities). This means the ID is scoped to a batch, i.e. it has the batch scope.

Note that the cards library is using the word "carousel" to refer to all of the card attachments contained in a single activity even if the channel doesn't support carousels or if the activity is not using the carousel attachment layout, because the term "activity ID" was already taken. Also, the word "batch" is a term borrowed from Bot Builder v3 to just mean a collection of activities. These activities can be grouped together because they were all sent by the same call to `SendActivitiesAsync` or they can have some arbitrary grouping defined by the developer.

#### Setting data ID's

You can use the static "set" methods of the `DataId` class to insert data ID's into various objects. For example, using `DataId.SetInBatch` will put data ID's in every action in every card in every activity in a batch:

```c#
DataId.SetInBatch(activities);
```

By default, the set methods will only insert an action ID. If you want to use a different data ID scope, you can pass a `DataIdOptions` object to the method. The following code will put a card ID in every action in every card in an activity:

```c#
DataId.SetInActivity(activity, new DataIdOptions(DataIdScopes.Card));
```

The cards library generates random GUID-based ID's for you, and it makes sure different ID's are generated for different objects according to their scopes. For example, if you have two cards in a carousel and each card has two actions and you want to use the carousel, card, and action scopes, all four actions will have different action ID's in their action data but they'll all have the same carousel ID, and two card ID's will be generated so that different actions in the same card will have the same card ID but different actions in different cards will have different card ID's. Your code might look like this:

```c#
DataId.SetInCarousel(attachments, new DataIdOptions(new List<string>
{
    DataIdScopes.Action,
    DataIdScopes.Card,
    DataIdScopes.Carousel,
}));
```

And you might end up with the following four objects in the action data of your four actions (notice which ID's are the same and which are different):

```json
{
    "action": "action-f5becd5f-60ed-40af-8567-5c6404ea92ee",
    "card": "card-3f0818f0-0c6c-4b62-9736-39c024576bae",
    "carousel": "carousel-6f4223be-c370-480c-a7b0-fdcfa33878a2"
}
```

```json
{
    "action": "action-a8e13e43-a0d4-4a82-a64c-2962a2c26842",
    "card": "card-3f0818f0-0c6c-4b62-9736-39c024576bae",
    "carousel": "carousel-6f4223be-c370-480c-a7b0-fdcfa33878a2"
}
```

```json
{
    "action": "action-d28b669f-674a-4756-b1b5-000d8e156409",
    "card": "card-ad6cd62d-2de7-4849-96f6-beabc0beb466",
    "carousel": "carousel-6f4223be-c370-480c-a7b0-fdcfa33878a2"
}
```

```json
{
    "action": "action-cee648cb-2074-4d26-98bd-6b3fc1c59981",
    "card": "card-ad6cd62d-2de7-4849-96f6-beabc0beb466",
    "carousel": "carousel-6f4223be-c370-480c-a7b0-fdcfa33878a2"
}
```

If you want to use a specific ID instead of generating a random ID then you can use the `DataIdOptions.Set` method:

```c#
var options = new DataIdOptions();

options.Set(DataIdScopes.Batch, "My batch ID");
```

The `DataId` class has 16 static set methods including the three you've seen so far:

- `SetInBatch`
- `SetInActivity`
- `SetInCarousel`
- `SetInAttachment`
- `SetInAdaptiveCard`
- `SetInAnimationCard`
- `SetInAudioCard`
- `SetInHeroCard`
- `SetInOAuthCard`
- `SetInReceiptCard`
- `SetInSigninCard`
- `SetInThumbnailCard`
- `SetInVideoCard`
- `SetInSubmitAction`
- `SetInCardAction`
- `SetInActionData`

Internally, the cards library uses something it calls the [*card tree*](./mermaid-diagram.md) to facilitate recursion for operations like these. At the top you have batches and each batch has indexed activities and each activity has an `Attachments` property and the `Attachments` property has indexed attachments and each attachment has a `Content` property and so on. The card tree can be entered at any of those "nodes" and it will drill down into indexes and properties and sub-properties of objects until it reaches its "exit" node.

#### Tracking and disabling

Once you've set ID's in your action data, remember that you need to track them in card manager state as well in order to disable them. The `CardManager` class has 5 methods for disabling in this way:

- `EnableIdAsync` - Tracks or forgets depending on tracking style
- `DisableIdAsync` - Tracks or forgets depending on tracking style
- `TrackIdAsync` - Tracks an ID in card manager state
- `ForgetIdAsync` - Removes an ID from the ID's tracked in card manager state
- `ClearTrackedIdsAsync` - Forgets all ID's in card manager state

If you want to treat all ID's as disabled by default and you only want to enable specific ID's (such as in the most recently sent card), you should use `TrackingStyle.TrackEnabled`. This means you'd have to enable an ID before it can be used, like this:

```c#
await cardManager.EnableIdAsync(
    turnContext,
    new DataId(DataIdScopes.Action, actionId),
    TrackingStyle.TrackEnabled);
```

Calling `EnableIdAsync` with the `TrackEnabled` tracking style will call `TrackIdAsync` internally, and you can instead call `TrackIdAsync` directly if you like:

```c#
await cardManager.TrackIdAsync(
    turnContext,
    new DataId(DataIdScopes.Action, actionId));
```

You can disable an ID whenever you want, but a common case will be to disable an ID after an action is used so that it can only be used once. When you disable an ID, make sure you use the same tracking style that you use to enable them:

```c#
await cardManager.DisableIdAsync(
    turnContext,
    new DataId(DataIdScopes.Action, actionId),
    TrackingStyle.TrackEnabled);
```

Since calling `DisableIdAsync` with the `TrackEnabled` tracking style will call `TrackIdAsync` internally, you can instead call `ForgetIdAsync` directly if you like:

```c#
await cardManager.ForgetIdAsync(
    turnContext,
    new DataId(DataIdScopes.Action, actionId));
```

In some situations you may want to forget all ID's at once, such as if you want all previously sent cards to be disabled for each new turn. You can easily do this with `ClearTrackedIdsAsync`:

```c#
await cardManager.ClearTrackedIdsAsync(turnContext);
```

#### Saving and deleting

If you're using a channel that supports message updates then you can use a card manager to delete actions, cards, carousels, and batches. Because a carousel is the same thing as an activity in this context, if you delete an action or a card without deleting the carousel then the containing activity will have to be updated rather than deleted. For example, if an activity contains three card attachments then updating the activity so that it only contains two of those attachments will effectively delete the third card. In order to update previously sent activities with modifications like that, the activities must be saved in card manager state. The `CardManager` class provides the `SaveActivitiesAsync` method for this purpose:

```c#
await cardManager.SaveActivitiesAsync(turnContext, activities);
```

Just like with tracking and disabling, you will want to make sure your saved activities contain action data ID's. In this case, the ID's will be used to identify which activity an action came from. Once your bot receives the action, you can call `DeleteActionSourceAsync` like this:

```c#
await cardManager.DeleteActionSourceAsync(turnContext, DataIdScopes.Card);
```

It uses the phrase "delete action source" instead of "delete card" etc. because it can delete an action or a card or a carousel or a batch depending on the scope you pass to it. The method looks at the incoming activity in the turn context and determines if the activity contains action data and uses the data ID from the scope you specify to update or delete the associated activities accordingly.

Note that "deleting activities" in this context means actually removing them from the channel conversation on the client side. The cards library uses the term *unsave* to mean removing an activity from card manager state. If any activities are deleted when you call `DeleteActionSourceAsync` then they get unsaved automatically, but if you want to unsave an activity manually then you can use `UnsaveActivityAsync`:

```c#
await cardManager.UnsaveActivityAsync(turnContext, activityId);
```

#### Middleware

#### Behaviors

### Preserving Adaptive Card input values in Microsoft Teams

### Translation

### Miscellaneous