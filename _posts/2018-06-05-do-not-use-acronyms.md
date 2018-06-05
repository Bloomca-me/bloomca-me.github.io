---
layout: post
title: Do Not Use Acronyms
keywords: software development, communication, efficiency, programming, naming, seva zaikov, bloomca
---

We like to optimize for efficiency, and commonly used phrases are a perfect candidate for it – it takes less than a second, and you have a whole bunch of weird looking acronyms. But they are here to help – everybody knows what it means, and it saves us time and space (so we don't write "Modern Improvements Mission", just using "MIM". What about MIM?). And it does help! It works perfectly for _some_ time, until time passes and new people come – it is a delay-action bomb.

All companies have some institutional knowledge, and it makes perfect sense – there is some domain knowledge, company-specific projects, historical data. However, while all of these are usually captured and documented – not always, but usually it is considered a good practice to do so – acronyms are often emerged naturally (they are used to save time, sharing knowledge is not their goal!), but leak to all documents without their own explanation. The main problem happens only after significant amount of time, after company's growth, when it starts to have department-scoped acronyms and (which is much worse) project-related acronyms.

This all is fine while they are used _only_ inside their own worlds, but at some point somebody outside will need to get some information about this project. In the beginning everything is fine – since it is a rare event, people will help to ramp up and, in case they are annoyed enough and we are lucky, they will write something somewhere. Why they won't write it later? Department might change completely, project can be completed or cancelled, people will be transfered or quit, and all the knowledge will be spread across the whole organization (so it is extremely unlikely that someone will write something). Later examination, or just going through docs of different department/project might become a really hard task.

There is a famous email from Elon Musk with the subject line: Acronyms Seriously Suck:

<blockquote>
<p>
There is a creeping tendency to use made up acronyms at SpaceX. Excessive use of made up acronyms is a significant impediment to communication and keeping communication good as we grow is incredibly important. Individually, a few acronyms here and there may not seem so bad, but if a thousand people are making these up, over time the result will be a huge glossary that we have to issue to new employees. No one can actually remember all these acronyms and people don't want to seem dumb in a meeting, so they just sit there in ignorance. This is particularly tough on new employees.
</p>
<p>
That needs to stop immediately or I will take drastic action - I have given enough warning over the years. Unless an acronym is approved by me, it should not enter the SpaceX glossary. If there is an existing acronym that cannot reasonably be justified, it should be eliminated, as I have requested in the past.
</p>
<p>
For example, there should be not "HTS" [horizontal test stand] or "VTS" [vertical test stand] designations for test stands. Those are particularly dumb, as they contain unnecessary words. A "stand" at our test site is obviously a test stand. VTS-3 is four syllables compared with "Tripod", which is two, so the bloody acronym version actually takes longer to say than the name!
</p>
<p>
The key test for an acronym is to ask whether it helps or hurts communication. An acronym that most engineers outside of SpaceX already know, such as GUI, is fine to use. It is also ok to make up a few acronyms/contractions every now and again, assuming I have approved them, e.g. MVac and M9 instead of Merlin 1C-Vacuum or Merlin 1C-Sea Level, but those need to be kept to a minimum.
</p>
</blockquote>
