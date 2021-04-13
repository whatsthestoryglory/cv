---
title: "Dynamic Playlist Generator"
date: 2021-04-09T12:29:25-05:00
description: Using SQL to create the ultimate personal radio station
hero: /images/posts/writing-posts/Idea.svg
menu:
  sidebar:
      name: SQLPlaylist
      identifier: sqlplaylist
      weight: 15
---

## The Problem

I have been known to have something of an eclectic music taste.  I can go from `TOOL` to `Billy Eilish` to `Big K.R.I.T.` to `Nelly Furtado` to `Armin van Buuren` without much consideration.  While I occasionally end up going down a bit of a genre rabbit hole from time to time, generally I like the mix.  I get bored if I listen to too much of the same genre.
With that said, I find most ways of having music picked for me to be frustrating.  Listening to the radio you end up only hearing top singles from a generally narrow genre band, so you're changing stations often.  The same thing happens with most of the dynamic playlists I've experienced from Spotify and similar streaming services - they take an artist or two that you're in to and play you a bunch of music that sounds like them.  It just always feels too narrow to me, and just because I like `Sigrid` and `Lorde` doesn't mean I'm interested in `Katy Perry` and `Justin Bieber`, you know?

## The Fix

### First Step

I have a large library of music at home and I use the `Logitech Media Server` (also known as `SqueezeboxServer`) to manage and play it across all my devices.  This allows me to do cool things like play the same music, synchronized, in each room of the house, without being locked into a specific vendor or format like Airplay.  This software has a robust selection of features and third party plugins that enable it to do just about anything I could possibly need.

I started by rating my music.  With over 25,000 tracks in the library, this is a long, boring, and time consuming process, but it needs to be done.  I created a scheme for myself to help with the ratings.

|Rating  | Meaning  |
|--:|---|
| 5☆  | I pretty much always want to listen to this.  `DNA` by `Kendrick Lamar` fit in here. |
| 4.5☆  | Great music, but maybe not every day.  `Time` by `Jungle` is one example for me on this one. |
| 4☆  | A little less than 4.5☆ but still music I know well and enjoy.  `Bullet with Butterfly Wings` by `The Smashing Pumpkins`  |
| 3.5☆  | Music I enjoy when it comes on the radio, but don't go out of my way to put on.  `Just a Girl` by `No Doubt`  |
| 3☆  | Music that's adjacent to the higher ☆ levels, but isn't otherwise a big draw for me.  `Don't Carry it All` by `The Decemberists`  |
| 2.5☆  | Other music that doesn't stand out as bad. `Everything Will Be Alright` by `The Killers`  |
| <2.5☆  | Music I have because it's a part of an album, but don't want to listen to randomly  |

### Second Step

Next I needed to find a way to generate playlists from this data.  I started by installing the `TrackStat` and `Dynamic Playlists` plugins for `Logitech Media Server`, which was great in that it gave me a way to play a dynamic playlist based on the ratings.  Unfortunately, it didn't give me much flexibility beyond selecting a rating I wanted to listen to, or just saying I want to listen to something like the "Top rated not played recently."  This didn't quite fit the bill for me, I want a wider experience than that.

In my younger years I would listen to the `A State of Trance` podcast by `Armin van Buuren` quite regularly.  One of the things that always stood out to me, both on this podcast itself but also in DJ sets overall, was that while I would enjoy the set overall, not every song was a 'banger'.  In fact, when there would be a set made up of 'top songs' I would generally tune out, it was too much.  Sets needed breathing room.  You have a really exciting track or two that pulls the crowd in and gets the energy up, and then you lighten up or slow down a little to give everybody a breather before bringing the energy back.  This is what I wanted from my playlists, to have lots of great music but intersperse it with unfamiliar or 'less great' music so it didn't get tiring.  This also gives the benefit of helping to explore music I don' know as well, without getting too tired of completely unknown music.  In comes the `SQL Playlist` plugin.

### SQL Playlist Plugin

In short, this plugin does exactly what it says, lets you use custom SQL for generating playlists.  It comes with several defaults to try and a creator to make some of your own if you don't know SQL yourself.  By using this I can create a single playlist that meets all of my needs by combining ratings, how recently a song was played, and how recently a song was added, and every time I click it I will end up with a completely unique playlist of music I love, like, sort-of know, and have yet to hear, mixed together.

### Generating the SQL

I started by creating one of the predefined playlist types and reviewing the SQL.  This gives me a starting point showing what columns I can use and what needs to be returned for the playlist to work.

```
-- PlaylistName:Cams Playlist Generator
-- PlaylistGroups:
create temporary table randomweightedratingslow as select tracks.url as url from tracks
	left join dynamicplaylist_history on
		tracks.id=dynamicplaylist_history.id and dynamicplaylist_history.client='PlaylistPlayer'
	left join track_statistics on
		tracks.url=track_statistics.url
	where
		audio=1
		and track_statistics.rating<50
		and dynamicplaylist_history.id is null
		and ifnull(track_statistics.lastplayed,0)<(unix_timestamp()-86400)
	order by random()
	limit 30;


create temporary table randomweightedratingshigh as select tracks.url as url from tracks
	left join dynamicplaylist_history on
		tracks.id=dynamicplaylist_history.id and dynamicplaylist_history.client='PlaylistPlayer'
	left join track_statistics on
		tracks.url=track_statistics.url
	where
		audio=1
		and track_statistics.rating>=50
		and dynamicplaylist_history.id is null
		and ifnull(track_statistics.lastplayed,0)<(unix_timestamp()-86400)
	order by random()
	limit 70;

create temporary table randomweightedratingscombined as 
	SELECT * FROM randomweightedratingslow 
	UNION 
	SELECT * from randomweightedratingshigh;

SELECT * from randomweightedratingscombined 
	ORDER BY random() 
	limit 10;

DROP TABLE randomweightedratingshigh;
DROP TABLE randomweightedratingslow;
DROP TABLE randomweightedratingscombined;
```

This is a perfect example for my needs.  Basically what this SQL does is it creates two separate lists; the first pulls 30 low-rated tracks and the second pulls 70 high-rated tracks; then combines them to make a 100 track list and pulls 10 at random from it.  Perfect, this is exactly what I was going for.  Now to create my own version.

## My Custom Playlist

First, lets get the 5☆ tracks.  You'll remember I described these as music I pretty much always want to listen to.  Let's grab 10 of the ones that haven't been listened to in 24hrs.

```
create temporary table fivestartracks as select tracks.url as url from tracks
	left join dynamicplaylist_history on
		tracks.id=dynamicplaylist_history.id and dynamicplaylist_history.client='PlaylistPlayer'
	left join track_statistics on
		tracks.url=track_statistics.url
	where
		audio=1
		and track_statistics.rating=100
		and dynamicplaylist_history.id is null
		and ifnull(track_statistics.lastplayed,0)<(unix_timestamp()-86400)
	order by random()
	limit 10;
```

Next, let's broaden the allowable ratings, but push out the last listened to be 3 days.

```
create temporary table fourhalfstartracks as select tracks.url as url from tracks
	left join dynamicplaylist_history on
		tracks.id=dynamicplaylist_history.id and dynamicplaylist_history.client='PlaylistPlayer'
	left join track_statistics on
		tracks.url=track_statistics.url
	where
		audio=1
		and track_statistics.rating>=90
		and dynamicplaylist_history.id is null
		and ifnull(track_statistics.lastplayed,0)<(unix_timestamp()-259200)
	order by random()
	limit 10;
```

And again once more, further increasing the last listened limit and reducing the minimum rating limit.

```
create temporary table fourstartracks as select tracks.url as url from tracks
	left join dynamicplaylist_history on
		tracks.id=dynamicplaylist_history.id and dynamicplaylist_history.client='PlaylistPlayer'
	left join track_statistics on
		tracks.url=track_statistics.url
	where
		audio=1
		and track_statistics.rating>=80
		and dynamicplaylist_history.id is null
		and ifnull(track_statistics.lastplayed,0)<(unix_timestamp()-604800)
	order by random()
	limit 10;
```

Now let's open it up wide, collect everything above 2.5☆

```
create temporary table twopointfiveuptracks as select tracks.url as url from tracks
	left join dynamicplaylist_history on
		tracks.id=dynamicplaylist_history.id and dynamicplaylist_history.client='PlaylistPlayer'
	left join track_statistics on
		tracks.url=track_statistics.url
	where
		audio=1
		and track_statistics.rating>=50
		and dynamicplaylist_history.id is null
		and ifnull(track_statistics.lastplayed,0)<(unix_timestamp()-604800)
	order by random()
	limit 10;
```

Music that I've recently added or modified is generally going to be something I'm interested in hearing.  Let's add some recently modified to the mix.

```
create temporary table recentlymodifiedtracks as select tracks.url as url from tracks
	left join dynamicplaylist_history on
		tracks.id=dynamicplaylist_history.id and dynamicplaylist_history.client='PlaylistPlayer'
	left join track_statistics on
		tracks.url=track_statistics.url
	where
		audio=1
		and track_statistics.rating>=50
    and track_statistics.added>STRFTIME("%s",DATE('NOW','-30 DAY'))
		and dynamicplaylist_history.id is null
		and ifnull(track_statistics.lastplayed,0)<(unix_timestamp()-86400)
	order by random()
	limit 10;
```

Then, just like the original example, I'm going to join all of these together to create a single table, and then select 50 tracks total at random from it.

```
-- PlaylistName:Random rated songs
-- PlaylistOption Unlimited:1
-- PlaylistGroups:
create temporary table fivestartracks as select tracks.url as url from tracks
	left join dynamicplaylist_history on
		tracks.id=dynamicplaylist_history.id and dynamicplaylist_history.client='PlaylistPlayer'
	left join track_statistics on
		tracks.url=track_statistics.url
	where
		audio=1
		and track_statistics.rating=100
		and dynamicplaylist_history.id is null
		and ifnull(track_statistics.lastplayed,0)<(unix_timestamp()-86400)
	order by random()
	limit 10;

create temporary table fourhalfstartracks as select tracks.url as url from tracks
	left join dynamicplaylist_history on
		tracks.id=dynamicplaylist_history.id and dynamicplaylist_history.client='PlaylistPlayer'
	left join track_statistics on
		tracks.url=track_statistics.url
	where
		audio=1
		and track_statistics.rating>=90
		and dynamicplaylist_history.id is null
		and ifnull(track_statistics.lastplayed,0)<(unix_timestamp()-259200)
	order by random()
	limit 20;

create temporary table fourstartracks as select tracks.url as url from tracks
	left join dynamicplaylist_history on
		tracks.id=dynamicplaylist_history.id and dynamicplaylist_history.client='PlaylistPlayer'
	left join track_statistics on
		tracks.url=track_statistics.url
	where
		audio=1
		and track_statistics.rating>=80
		and dynamicplaylist_history.id is null
		and ifnull(track_statistics.lastplayed,0)<(unix_timestamp()-604800)
	order by random()
	limit 20;

create temporary table twopointfiveuptracks as select tracks.url as url from tracks
	left join dynamicplaylist_history on
		tracks.id=dynamicplaylist_history.id and dynamicplaylist_history.client='PlaylistPlayer'
	left join track_statistics on
		tracks.url=track_statistics.url
	where
		audio=1
		and track_statistics.rating>=50
		and dynamicplaylist_history.id is null
		and ifnull(track_statistics.lastplayed,0)<(unix_timestamp()-604800)
	order by random()
	limit 10;

create temporary table recentlymodifiedtracks as select tracks.url as url from tracks
	left join dynamicplaylist_history on
		tracks.id=dynamicplaylist_history.id and dynamicplaylist_history.client='PlaylistPlayer'
	left join track_statistics on
		tracks.url=track_statistics.url
	where
		audio=1
		and track_statistics.rating>=50
    and track_statistics.added>STRFTIME("%s",DATE('NOW','-30 DAY'))
		and dynamicplaylist_history.id is null
		and ifnull(track_statistics.lastplayed,0)<(unix_timestamp()-86400)
	order by random()
	limit 10;

create temporary table allselectionscombined as 
	SELECT * FROM fivestartracks 
	UNION 
	SELECT * FROM fourhalfstartracks
  UNION
  SELECT * FROM fourstartracks
  UNION
  SELECT * FROM twopointfiveuptracks
  UNION
  SELECT * FROM recentlymodifiedtracks ;

SELECT * from allselectionscombined 
	ORDER BY random() 
	limit 50;

DROP TABLE fourhalfstartracks;
DROP TABLE twopointfiveuptracks;
DROP TABLE recentlymodifiedtracks;
DROP TABLE fivestartracks;
DROP TABLE allselectionscombined;
```

There we go, now I have a playlist generator that builds playlists dynamically based on ratings I set up, what I have listened to recently, and what I have added recently.

### What about my wife?

My wife and I have very similar music tastes, though she tends to find I listen to a *little too much* rap and eletronic music.  So, I created a variant of my playlist generator that would work for her.

To make this as easy as possible, I created another temporary table called reducedtracks that would select all tracks *except* those labelled with Rap, Hip Hop, and related genres.  I then modified the subsequent tables to pull from reducedtracks instead of tracks and everything else *just worked*.

```
create temporary table reducedtracks as select * from tracks
  join genre_track ON 
    tracks.id=genre_track.track
      join genres ON genres.id = genre_track.genre AND
      genres.name<>'Trance' AND 
      genres.name<>'Electronic' AND 
      genres.name<>'Rap' AND
      genres.name<>'Hip Hop' AND
      genres.name<>'Hip-Hop' AND
      genres.name<>'Rap/Hip-Hop' AND
      genres.name<>'Rap/Hip Hop' AND
      genres.name<>'Gangsta Rap' ;
```

## Final Output

Now let's check our work and take a look at what a sample playlist might look like

| Song Title         | Artist                                                      | Album Title             |
|--------------------|-------------------------------------------------------------|-------------------------|
| Magpie             | Caribou                                                     | Suddenly                |
| Good Ass Intro     | Chance the Rapper feat. BJ the Chicago Kid                  | Acid Rap                |
| Like Toy Soldiers  | Eminem                                                      | Encore                  |
| 00000 Million      | Bon Iver                                                    | 22, a Million           |
| That Song          | Big Wreck                                                   | Big Shiny Tunes 3       |
| In the City        | Eagles                                                      | Hell Freezes Over       |
| Streets on Fire    | Lupe Fiasco                                                 | Lupe Fiasco’s The Cool  |
| LOVE.              | Kendrick Lamar feat. Zacari                                 | DAMN.                   |
| HEAD SHOTS         | Tobe Nwigwe feat. D Smoke                                   | CINCORIGINALS           |
| Electric Feel      | MGMT                                                        | Oracular Spectacular    |
| We Go On           | The Avalanches feat. Cola Boyy & Mick Jones                 | We Will Always Love You |
| Sunny's Time       | Caribou                                                     | Suddenly                |
| Spill Vill         | Spillage Village, Desi Banks & Big Rube feat. Kountry Wayne | Spilligion              |
| Moon Song          | Phoebe Bridgers                                             | Punisher                |
| Oblique City       | Phoenix                                                     | Bankrupt!               |
| Oh My Heart        | R.E.M.                                                      | Collapse Into Now       |
| Conventioneers     | Barenaked Ladies                                            | Maroon                  |
| 初登場             | UNKLE                                                       | Rōnin I                 |


Now I have an easy "Just Press Play" solution to listening to music around the house.  Every time I press play, something different will play, and it will lean towards music I like.  Perfect!