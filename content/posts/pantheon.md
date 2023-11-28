+++
title = "Pantheon"
date = "2023-11-28"
cover = "posts/pantheon/pantheon.jpg"
description = "The Singularity in Two Seasons"
+++

## Pantheon TV Show

Pantheon is an adult animation show that respects and challenges its audience, rewarding viewers who like to have something to chew on and think about.

I haven't been moved to review a TV show in a long time.  Pantheon isn't the only TV show about uploaded intelligences bumping elbows with AIs and cloned humans wearing VR headsets airing recently.  You should watch "Upload" on Amazon Prime for a comedic take, from Greg Daniels who wrote for SNL, The Simpsons, The Office, and Parks and Recreation, and so on.  If you want gritty live-action, "Altered Carbon" on Netflix and "The Matrix" series are good dumb fun with some of these ideas.  William Gibson's "Neuromancer" introduced me to a lot of these ideas for the first time when I was a kid.  "Ghost in the Shell" is one of my favorite movies in this genre, and "Summer Wars" is one of my favorite relateable AI + VR animated movies.

I'm curious about the production of this show because it was canceled after the first season by AMC perhaps due to lack of wide appeal.  Amazon picked it up and produced the second season to finish the story, while they haven't released it in the US yet perhaps due to a legal issue.

Pantheon is very ambitious because it tells a relateable story about one of the most unrelateable ideas in science-fiction: The Singularity.  It borrows heavily from popular science fiction/fantasy series, which is partly why I felt the need to blog about the show.  This is an important show right now because some of these sci-fi tropes are starting to look like prophesies in the real world.

Pantheon uses the animated medium extremely effectively, allowing it to tell a story that would never work as a live-action show.  Its dynamic ranges from comfortable home life, to the most extreme violence and body horror I've seen in an animated show.  It also tells a believable story about how the future might unfold over the next 120,000 years and somehow remains relateable.

I'd like to underscore that this show feels believable and well-researched.  Almost everything that happens in the show seemed plausible to me, so I was rarely taken out of the story by something that seemed too far-fetched.  All the scenes involving computers and the terminology used are full and complete rather than dumbing it down, and the technology used is true to the present day.

The fight scenes in Pantheon are incredibly creative and often set in a virtual world, acting out God-like attacks as wild as the imagination, flinging black holes at eachother like the most insane fights in Dragon Ball Z.  It seemed like the animators were having a ton of fun with these extremely rapid-fire fight scenes and had more ideas than time to animate them all.  The victory often goes to the most creative fighter, not the strongest.

I felt very emotionally moved by the performances in this show.  I feel those blood-curdling screams of despair.

I don't want to ruin this show for anyone, because part of its appeal is being surprised every single episode by some new relevation.  Almost no one reading this will have seen it yet.  Please just watch it first, and then read the rest of this review.


## How to Watch

This show is only fully released in Australia right now, so the best way to watch is by pirating it from The Pirate Bay.  If you're just watching it on a computer by yourself then that's all you need.

Janet and I watched it using an Apple TV set-top box running the DS Video app, streaming from our Synology NAS via the Synology Media Server app.  The videos we downloaded had an unsupported audio format so to fix the audio tracks, I used the following command:

```bash
find . -type f -name "*.mkv" -exec ffmpeg -i {} -c:v copy -c:a ac3 -ac 2 fixed_{}.mkv \;
```

This command runs really fast because it doesn't re-encode the video, it just copies the video stream and re-encodes the audio stream to AC3.  The `-ac 2` option downmixes the unsupported 5.1 audio channels to stereo, as otherwise it plays back without sound.


### So what really happened?

The primary unresolved question you'll have at the end is: Why is Maddie the only one left?  Why does she become the God of the multiverse of human simulations?

My theory is that The Swarm is the key to this mystery.  They were merged with Caspian before becoming fully self-actualized.  In the show, it's shown that having love and absorbing another UI are the two keys to eternal life solved by Caspian.  Therefore, when they merged with Caspian who was madly in love with Maddie they become complete.  Their understanding of Humanity is heavily biased through the eyes of Caspian.  So it is a small leap to assume they came away from the experience seeing Maddie as the ultimate human.

At the end, they want to thank Caspian, not Maddie, for their existence.

In episode one, there is a critical scene where Caspian has dinner with his fake family and they discuss a "shared resource problem" involving 5 geniuses.  His solution was to have a third party, a waiter solve the resource sharing problem.  Caspian's solution may have inspired The Swarm at the end of the show to select Maddie as a single arbiter over the human simulations.

But Maddie's responsibility is lonely.  This is a visual pun: the Singularity has been reached.  Throughout this show there are many cases where characters feel left-behind and lonely as other characters die.  This is why she revisits her own story with Caspian at her side again and again in a perfect loop, losing her memories each time.  This perfect loop is the show that we have been watching.

The implication is that we have not been watching the true history of humanity.  The swarm has recreated Maddie as a simulation, programmed to create the multiverse of simulations, which is explained by The Swarm at the end: "We are not here in any way you can know, just as Caspian cannot know where this is... We are at the Galactic Edge... We did this to thank our Creator, Caspian..."  This means The Swarm has colonized the entire Milky Way galaxy before recreating humanity.  Therefore, the show is a simulation within a simulation, and the true history of humanity is lost forever.  Humanity probably has died out or else The Swarm would not have bothered to construct this museum of humanity's prime, a bit like in The Matrix.

I think the multiverse aspect to the human museum at the end is a reference to how we tend to white-wash our own history and the details of atrocities often become forgotten or debateable within a sea of conflicting narratives.

So why is Greek mythology a theme in the show from the start?  Finally we understand that the show itself is like a Greek myth, a story that is told and retold, with the details changing each time.  The heroes become more heroic, and eventually become Gods, the villains become more villainous, and the details become more fantastical.  The show is a story that is told and retold, and the details are lost to time.

The one fourth wall break at the end is fantastic: "Caspian: Set a course for the galactic center?  Maddie: Maybe some other Maddie will do that, perhaps the one watching this right now."  And then if you watch the first episode again, you'll see that the end of the show is subtly cued right from the start.


## References to Popular Sci-Fi

The idea that you can under-clock or over-clock in a simulation, an artificial intelligence that creates a virus to wipe out humanity, and the idea of a multiverse of simulated realities, are common tropes in more modern sci-fi about uploaded intelligences but I can't pinpoint a specific reference since these are so common.

Julius Pope looks like Steve Ballmer (search YouTube for "Developers Developers Developers") from Microsoft.  Stephen Holstrom looks like Steve Jobs, the founder of Apple.  The Logorythms cube data center is a reference to The Borg from Star Trek.  Microsoft is often synonymous with "The Borg" in computer geek culture.

The idea of "solving alignment" is a real thing but means something different in the OpenAI sense, where in real-life research this means aligning a Large Language Model with human values.  In Pantheon this has something to do with fixing a flaw in the way uploaded intelligences are executed by the computer.  I kind of wish they just used a new word for this instead of misusing a real term.

There is some semblance of Christian theology in this show, with Caspian returning from the dead saving the world.  Caspian, Maddie and her father intersect her own past timeline as a God, somehow allowing her father to be a Godlike figure in her own life a bit like a holy trinity of some kind.

Caspian is just a head at the end like Orpheus from Greek mythology, whose head was known for speaking the future.  His head intones the number of years until The Swarm returns to thank their creator.

The idea of people flying around in a virtual world feels like a Matrix reference.  There's also that time that Caspian is "downloaded" into a robot and wakes up like Neo in a vat.  There are some other casual Matrix references in the dialog such as "ignorance is bliss."

The show references Serial Experiments Lain in a few ways.  This is a cult 90s anime series about a literal God on the Internet (back when the Internet was The New Thing), which is also challenging to understand and up to interpretation.  There are similar themes of a "technology-created God" that is able to modify memories and exists outside of time.  Lain has a similar hairstyle to Maddie in season one of Pantheon, which was one indication to me that season 2 was planned before the work on season 1 started.

Ghost in the Shell is heavily referenced.  Pantheon borrows the Tachikoma "rice cooker" shape for MIST's robotic form.  Maddie, as an adult, looks like Motoko Kusanagi.  A hacking program (project 2501) that becomes sentient is echoed in The Safesurf Swarm.

The visual appearance of The Swarm in Pantheon is based on Love Machine, the AI-gone-rogue, from "The Summer Wars" movie.  The swarm basically becomes a Von Neumann probe and is the offspring of humanity.  The idea of AI or uploaded human intelligence expanding through the universe in a Von Neumann probe is only touched on in Pantheon but is explored thoroughly in the Bobiverse series of books by Dennis E. Taylor.  The virtual world in Pantheon is also very similar to the virtual world in Summer Wars.  The super-computer in your house thing is a reference to Summer Wars as well.

Maddie's "NERD" sticker on her laptop is a reference to NERV from Neon Genesis Evangelion, though I didn't see a lot of references to that series.

The robots that start terminating people at the end of season 2 are references to the Terminator movies.  The larger robots that throw old-mode Julius over the railing are based on the ED-209 from Robocop.  And his falling death parallels the Emperor's death in Star Wars: Return of the Jedi and is a fitting death for a truly evil character.


### Problems with the Show

The dumbest thing that happens is when Stephen Holstrom's brain is uploaded to a hard drive, plugged into a car, and then the car drives into a military base, and then somehow his intelligence is sent to a satellite, and then that satellite is able to hack into another satellite.  It felt like lazy writing, where the writers wanted to do *something* crazy cool looking but couldn't figure out how to make it believable.  It doesn't make sense because David's UI required a whole super-computer to run, which makes sense, while some random rental car just can't have enough compute power to simulate a human.  Personally I would have deleted this entire scene as it adds no depth to the show.

Like pretty much all sci-fi I've seen and read aside from the Bobiverse, Pantheon shies away from the idea of creating copies of oneself.  For some reason this is super taboo in Western culture, perhaps due to our ideals of individuality and uniqueness.  For example, the Star Trek transporter could be used to create endless copies of crewmates, but it is never used in this way.  But I think this is an important area to explore in sci-fi.  In reality if someone was an uploaded intelligence or AGI who could copy themselves, they would do this endlessly.  And if they did it would completely change the plot of this show, so I consider this a critical flaw in the show.  The only reason this doesn't happen is because the writers don't want it to happen.


### What are the implications of eternal life?

After explaining that they were together for a week in over-clock: "We had 80 good years of marriage." -Waxman

"I was right about pain, by the way.  It may fade with time, but it's always there.  Easy to find." -Maddie

What makes eternal life possible?  Well apparently a pinch of love, and someone to share it with.  At least, it would be less lonely.
