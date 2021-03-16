---
title: Durable Entities + SignalR + React + MobX + TypeScript = Durable MVC
permalink: /2021/03/15/durable-mvc
---
![teaser]({{ site.url }}/images/durable-mvc/teaser1.png)
# Durable Entities + SignalR + React + MobX + TypeScript = Durable MVC.



*Quick shortcut to the template repo: [durable-mvc-starter](https://github.com/scale-tone/durable-mvc-starter)*

Although invented decades ago, the architectural pattern known as [Actor Model](https://en.wikipedia.org/wiki/Actor_model) is now having a re-birth. In this new serverless world 'classic' Object-Oriented Programming loses more and more of its power (when your class instances die too early to produce any significant work, what's the point of conceiving them?), so [Virtual Actors](https://www.microsoft.com/en-us/research/publication/orleans-distributed-virtual-actors-for-programmability-and-scalability/) now act as a modern equivalent to puny ephemeral classes from an early age. In Azure we now have [Azure Functions Durable Entities](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-entities?tabs=javascript) as yet another implementation of Virtual Actors, and it provides the advantage of writing your Actors in various programming languages, e.g. in TypeScript.

Out-of-the-box TypeScript support for Durable Entities is so far very limited (function-style only), so I started with implementing [a simple way to define your entities as TypeScript classes](https://github.com/scale-tone/durable-mvc-starter/blob/main/common/DurableEntity.ts).

A typical cloud-based service uses to have some sort of user interface, most typically browser-based and quite often implemented as a SPA (Single-Page App). Azure Functions to not have a built-in instrument for building and serving static HTML/JS/CSS files of your SPA ([Azure Static Web Apps](https://docs.microsoft.com/en-us/azure/static-web-apps/overview) do have that, but they are still in preview), so my next step was to add a client-side React-based project as [a subfolder of my main Azure Functions project](https://github.com/scale-tone/durable-mvc-starter/tree/main/ui), make both projects compile together and then add [a simple Function for serving the client-side static artifacts](https://github.com/scale-tone/durable-mvc-starter/blob/main/serve-statics/index.ts). Now the solution looks solid and can be built and run with a single hit of F5 button. Both projects are written in TypeScript, of course.

Now it would be nice to expose at least some of our Durable Entities to the client, so that the page can send signals to them (call their methods) and render their state. Entity's state is by definition frequently changing, so normally the client would need to periodically query for it, which is quite expensive both technically and financially. A much better idea would be to automatically propagate state changes to the client, and here comes [Azure SignalR Services](https://docs.microsoft.com/en-us/azure/azure-signalr/signalr-overview) to the rescue. So I added [a basic scaffolding for establishing authenticated SignalR connections](https://github.com/scale-tone/durable-mvc-starter/blob/main/negotiate-signalr/index.ts) and implemented incremental state transfer in form of [RFC 6902 (JSON Patch)](https://tools.ietf.org/html/rfc6902) documents.

Finally, since the client side is React-based, it would be very nice to have entity states being automatically rendered and re-rendered, once they change. This is absolutely possible to do thanks to [MobX](https://mobx.js.org/the-gist-of-mobx.html) library, which can turn any arbitrary JavaScript object into an [observable](https://mobx.js.org/observable-state.html). So I implemented [DurableEntitySet\<TState\> - a generic container for entity states](https://github.com/scale-tone/durable-mvc-starter/blob/main/ui/src/common/DurableEntitySet.ts), which exposes a collection of entities of certain type as an observable collection via its `items` property. Now we can wrap that collection with some JSX - and the UI will automatically refresh itself once a new entity is added or an existing entity is changed/removed (destroyed).

Combined all of this altogether, I can now present it to you in form of this GitHub template repo - [durable-mvc-starter](https://github.com/scale-tone/durable-mvc-starter). I even dare to codename this architectural approach 'Durable MVC'. Why? Because it looks like MVC ([Model-View-Controller](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller)), but instead of controllers the logic is implemented in form of Durable Entities.

History is indeed a spiral, so there is, of course, nothing fundamentally new in this architectural approach. Keeping the logic on server side and rendering the UI on the client - that's basically what all 'classic' web applications do (server-side code produces plain old HTML, which is then rendered by your browser), just having that logic implemented as a Durable Entity makes it survive underlying computing instance crashes and frees you from loading/persisting the state (and from synchronizing access to it). Pushing state changes to the client via some notification mechanism (SignalR, [socket.io](https://socket.io), plain [WebSockets](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API), etc) is also quite typical, though it seems more efficient to do that incrementally (send diffs, not entire objects). And the idea of [binding](https://docs.microsoft.com/en-us/windows/uwp/data-binding/data-binding-and-mvvm) the UI markup to some underlying View Model and having the UI automatically re-rendered upon View Model changes is the basic idea of widely adopted [MVVM (Model-View-ViewModel) architectural pattern](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93viewmodel), and this is what [MobX](https://mobx.js.org/the-gist-of-mobx.html) implements for React. So now it was just a matter of aggregating all of these ideas in one single solution and have them implemented in one same programming language - TypeScript.

So, the [durable-mvc-starter](https://github.com/scale-tone/durable-mvc-starter) template repo is intended to be the starting point for further development. Technically it is a pre-configured [Azure Functions Node.js project](https://docs.microsoft.com/en-us/azure/azure-functions/functions-reference-node?tabs=v2#typescript), so you can run, deploy and add more Azure Functions to it in the same way as you would normally do. It also includes this [simple sample CounterEntity](https://github.com/scale-tone/durable-mvc-starter/blob/main/CounterEntity/index.ts), to demonstrate the idea. You can start by running the project locally and playing with that entity in your browser (statics are by default served under root URL, so you just need to navigate to http://localhost:7071).

More examples you can find in this separate [durable-mvc-samples](https://github.com/scale-tone/durable-mvc-samples) repo. Let's take a closer look to [appointments-sample](https://github.com/scale-tone/durable-mvc-samples/tree/main/appointments-sample), which is an app that allows users to create and share appointments (and accept or decline them). Here is the [appointment's state](https://github.com/scale-tone/durable-mvc-samples/blob/main/appointments-sample/ui/src/shared/AppointmentState.ts):
```
export enum AppointmentStatusEnum {
    Pending = 0,
    Accepted,
    Declined
}

export class AppointmentState
{
    participants: { [name: string]: AppointmentStatusEnum } = {};
    status: AppointmentStatusEnum = AppointmentStatusEnum.Pending;
}
```
It only contains a dictionary of invited participants and an overall appointment's status. 
Now let's look at [AppointmentEntity](https://github.com/scale-tone/durable-mvc-samples/blob/main/appointments-sample/AppointmentEntity/index.ts)'s code:
```
class AppointmentEntity extends DurableEntity<AppointmentState>
{
    // Initializes new appointment instance
    init(participants: string[]) {

        // Adding appointment creator to the list of participants
        if (!participants.includes(this.stateMetadata.owner)) {
            participants.push(this.stateMetadata.owner);
        }

        this.state.participants = {};
        for (var name of participants) {
            this.state.participants[name] = AppointmentStatusEnum.Pending;
        }
        this.state.status = AppointmentStatusEnum.Pending;

        // Sharing this entity with all participants
        this.stateMetadata.allowedUsers = participants;
    }

    // Handles someone's response
    respond(isAccepted: boolean) {

        // Saving user's response
        this.state.participants[this.callingUser] = isAccepted ? AppointmentStatusEnum.Accepted : AppointmentStatusEnum.Declined;

        // Updating appointment's status
        if (Object.keys(this.state.participants).every(name => this.state.participants[name] === AppointmentStatusEnum.Accepted)) {
            // Everyone has accepted
            this.state.status = AppointmentStatusEnum.Accepted;
        } else if (!isAccepted) {
            // Someone has declined
            this.state.status = AppointmentStatusEnum.Declined;
        }
    }

    // Deletes this entity
    delete() {
        this.destructOnExit();
    }

    // Overriding visibility. This entity should only be visible to people mentioned in this.stateMetadata.allowedUsers
    protected get visibility(): VisibilityEnum { return VisibilityEnum.ToListOfUsers; }
}
```
The entity has three methods (signal handlers): `init()` initializes a new appointment with participants, `respond()` handles responses from participants and `delete()` destructs this entity instance. Note that it also overrides the `visibility` property to set it to `VisibilityEnum.ToListOfUsers`, so that the appointment is only accessible by its participants and by nobody else (all currently supported levels of visibility you can find [here](https://github.com/scale-tone/durable-mvc-starter/blob/main/common/DurableEntity.ts#L11)).
Now on the client we just need to instantiate an observable collection of these entities:
```
    appointments: new DurableEntitySet<AppointmentState>('AppointmentEntity')
```
and then bind some JSX markup to it. Not going to post the markup here, but you can quickly find it [in the sample codebase](https://github.com/scale-tone/durable-mvc-samples/blob/main/appointments-sample/ui/src/App.tsx#L48). Just note that the React component needs to be marked as [observer](https://mobx.js.org/react-integration.html), so that it knows how and when to re-render itself.

There's, of course, absolutely no need to adopt all the features of [durable-mvc-starter](https://github.com/scale-tone/durable-mvc-starter) at once. E.g. you might only be interested to have your server-side and client-side code in one project and have the statics automatically served - then just remove unneeded code and go ahead. Or you might want to start by defining your Durable Entities with class-based syntax and not use client-side features - that's perfectly fine as well, and for server-side type-safe interaction with your entities you can use this [DurableEntityProxy\<TEntity\>](https://github.com/scale-tone/durable-mvc-starter/blob/main/common/DurableEntityProxy.ts) helper.

Now let's discuss it all [via Twitter](https://twitter.com/tino_scale_tone), as I'm very much interested in any kind of feedback and especially in any kind of contribution to this project.