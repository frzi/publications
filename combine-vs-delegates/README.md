> Originally published on [Medium](https://medium.com/@frzi/using-combine-instead-of-delegates-or-passing-through-a-different-way-of-thinking-ea376849db14), December 22, 2022

# Using Combine to replace delegates and notifications — or: Passing through a different way of thinking

![Header](combine-hero-image2-cropped.jpg)

The delegation pattern can be found everywhere in Apple's frameworks. You've most likely used it countless times if you've developed iOS or macOS apps. Let's go over it real quick for the sake of this article.

You have an object (the sender) that references another object (the delegate):

```swift
class AudioPlayer {
    weak var delegate: AudioPlayerDelegate?
}
```

You have a protocol for said delegate:

```swift
protocol AudioPlayerDelegate: AnyObject {
    func audioPlayerConnected(_ audioPlayer: AudioPlayer)
    func audioPlayer(_ audioPlayer: AudioPlayer, changedSong song: Song)
    // Etc...
}
```

And at some point the sender object interacts with the delegate. For example to inform a certain event has occurred:

```swift
delegate?.audioPlayerConnected(self)
```

This is all easy to understand and it has served us well for decades.

Now let’s look at a different implementation to achieve something similar.

In 2019 Apple introduced [Combine](https://developer.apple.com/documentation/combine). A powerful framework that allows for reactive programming with a stream-like interface. One particularly interesting object in Combine we’ll be focusing on is [`PassthroughSubject`](https://developer.apple.com/documentation/combine/passthroughsubject). It's a publisher that simply passes through a value to its subscribers when you send it one. That's it. And as it turns out it makes for a great way to define events!

Here’s what it could look like:

```swift
class AudioPlayer {
    let onConnected = PassthroughSubject<Void, Never>
    let onSongChanged = PassthroughSubject<Song, Never>
}
```

The sender object no longer contains a `weak var delegate` property. Nor is there a delegate protocol. Rather, the object exposes its events as `PassthroughSubject` publishers. Each defined with the type of value it passes through (using `Void` if there is no value), and has `Never` for `Failure`: meaning the publisher will never send an error. This makes subscribing to the publisher a whole lot easier.

Objects can subscribe to these events wherever and whenever they wish, just like you would with any other Combine publisher:

```swift
private var cancellables = Set<AnyCancellable>()

audioPlayer
    .onSongChanged
    .sink { song in
        print("Current song is", song.title)
    }
    .assign(to: &cancellables)
```

When the event in question occurred the object can inform all the subscribers using `send()`:

```swift
onSongChanged.send(newSong)
```

> Use a tuple to send multiple values at once:
> `let onUserRequest = PassthroughSubject<(User, Song), Never>()`

That’s the gist of it. It’s clearly a different way of thinking and programming, and if you’re unfamiliar with this pattern it may take a second to fully wrap your head around it.

> Notice how the names of these `PassthroughSubjects` start with 'on'. This is to better differentiate the events from all other properties of the object. > A clear indication this is an 'event'.

Needless to say, let’s go through the ups and downs of using this pattern.

---

## Unlimited subscribers
With the delegation pattern you tend to have only one of them:

```swift
weak var delegate: AudioPlayerDelegate?
```

But this can be pretty limiting. Now only one object has the privilege of listening to events.

To combat this developers have either resorted to `NotificationCenter` (which is both cumbersome and **not** type safe!) or have come with their own custom solution:

```swift
public var listeners = Set<WeakValue<AudioPlayerDelegate>>()

// Adding a listener:
audioPlayer.listeners.insert(WeakValue(self))

// `AudioPlayer` informing all listeners:
listeners.forEach { $0.value?.audioPlayerConnected(self) }
```

Which is cute and all, but rather unstandardized and unfamiliar to anyone but themself.

Combine brings Swift users a well documented and, dare we say, standardized way of writing reactive code. No need to reinvent the wheel.

## Subscribe only to what you care about
Delegates work with protocols:

```swift
protocol AudioPlayerDelegate: AnyObject {
    func audioPlayerConnected(_ audioPlayer: AudioPlayer)
    func audioPlayerDisconnected(_ audioPlayer: AudioPlayer)
    func audioPlayer(_ audioPlayer: AudioPlayer, changedSong song: Song)
    func audioPlayer(_ audioPlayer: AudioPlayer, changedQuality quality: Quality)
    // Etc...
}
```

The protocol defines the interface — i.e. the required methods and properties — of an object. However, sometimes we don’t care about all the events. We just want to listen to one event. And… well you can see where this is going. This could mean you’ll have objects containing empty methods or properties when conforming to a protocol:

```swift
class NowPlayingUI: AudioPlayerDelegate {
    func audioPlayer(_ audioPlayer: AudioPlayer, changedSong song: Song) {
        albumArt.image = UIImage(contentsOf: song.albumArt.path)
        titleLabel.text = song.title
    }

    // We don't care about these. But Swift demands we include them.
    func audioPlayerConnected(_ audioPlayer: AudioPlayer) {}
    func audioPlayerDisconnected(_ audioPlayer: AudioPlayer) {}
    func audioPlayer(_ audioPlayer: AudioPlayer, changedQuality quality: Quality) {}
}
```

One horrible way to circumvent this is to use the disgusting `@obj optional` keywords. But as Swift grows we should rely less on these nasty ugly legacy Objective-C features.

With Combine other objects are able to subscribe only to whatever event they care about.

With that all said and done let’s look at some of the challenges when using `PassthroughSubjects` for events...

## Always having to use `[weak self]`
Every time you use `.sink` on a publisher, and you reference self in any way, you're almost always forced to use `[weak self]`.

```swift
audioPlayer
    .onSongChanged
    .sink { [weak self] song in
        self?.songTitle = song.title
        self?.songDuration = Double(song.duration) / 1000
    }
    .store(in: &cancellables)
``` 

Okay — so you’re not forced per se, but it is safer. This is of course to prevent circular references and to prevent keeping objects in memory beyond their lifespan. Using `[weak self]` can be tedious as you then have to work with an optional `self`. However, if you're feeling courageous you can use `[unowned self]`! (But seriously though, don't...)

## One-way communication
The original delegation pattern allows for two-way communication between the sender and the delegate, if so desired. This allows the sender to ask the delegate to return or modify a value. Or to tell the sender how to respond to a certain event.

```swift
// Asking the delegate for a value.
quality = delegate?.audioPlayerPreferredQuality(self) ?? .auto

// Asking the delegate whether it's okay to proceed.
guard delegate?.audioPlayer(self, allowStream: url) != false else {
    return
}
```

Two-way communication is not Combine’s intended purpose!

## Apple-bound (-ish)
Combine is a framework developed by Apple *for* Apple platforms. It’s closed source and only works on iOS (iPadOS), macOS, tvOS and watchOS. Which means using Combine can be a bit of a challenge if you were using Swift for Linux or Windows projects.

Luckily there are open source alternatives — like [CombineX](https://github.com/cx-org/CombineX) and [OpenCombine](https://github.com/OpenCombine/OpenCombine) — that strive to become a perfect replica of Apple’s Combine framework. Except open source and not bound to Apple platforms.

## Conclusion
Combine is an extremely powerful and convenient framework. It playing a prominent role in SwiftUI means a lot of developers will become familiarized with it (many already have). Meaning there’s no need to reinvent the wheel yourself. Using `PassthroughSubject` for events will allow you (and perhaps other developers) to further harnass the power of Combine.

Nonetheless, as shown above, it is not necessarily a full replacement for the delegation pattern. It’s important to fully understand the flow of your code before deciding to go with `PassthroughSubject`s (that is, if this article has you convinced!).

For me personally however, I can say I’ve written far less delegate and NotificationCenter code since the introduction of Combine. 😉

Thanks for reading!

---

## Extra
If `PassthroughSubject<T, Never>` is too much of a mouthful for you then consider using a typealias to make things slightly more convenient:

```swift
typealias Event<T> = PassthroughSubject<T, Never>

let onStart = Event<Void>()
let onSongChanged = Event<Song>()
```