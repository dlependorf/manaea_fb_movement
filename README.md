
Sean Manaea has had an up and down 2020, and there has been a lot of
talk lately around his fastball velocity and how it seems to track
closely with how well he pitches. A pitcher’s velocity is definitely
important, but I don’t think the conversation on Manaea has hinged
around the right thing. Instead of fastball velocity, we should be
talking more about fastball movement.

If you want to follow along, all R code needed to collect, transform,
and analyze the data (and even build this post\!) is in the [R Markdown
file in the GitHub repository attached to this
page](https://github.com/dlependorf/manaea_fb_movement/blob/master/README.Rmd).

## About Sean Manaea and his shoulder

Sean had shoulder surgery at the end of 2018, and at the time, [Jane Lee
of MLB.com reported
this](https://www.mlb.com/news/a-s-sean-manaea-has-shoulder-surgery-c295372404):

> Manaea, \[A’s head trainer\] Paparesta said, was dealing with chronic
> shoulder impingement for two years, worsening to the point where he
> couldn’t pitch without pain. The 26-year-old lefty was 12-9 with a
> 3.59 ERA in 27 starts this season, twirling a no-hitter against the
> Red Sox on April 21.

In theory, this means that Sean Manaea was healthy in 2016, and his
shoulder started acting up in 2017 and 2018. If that’s true, we might be
able to see this in his how his fastball behaved across those years.

## Grabbing the data

To grab the pitch tracking data needed for this, I’m going to use Bill
Petti’s fantastic [baseballr](https://github.com/BillPetti/baseballr) R
package, which comes with functions that grab raw pitch data from MLB
via Baseball Savant. I’m also loading the
[tidyverse](https://www.tidyverse.org/) suite of R packages. Note that
as I write this, baseballr is not yet on the CRAN repository of R
packages, so you’ll have to install it directly from its GitHub repo
with `remotes::install_github("BillPetti/baseballr")`.

``` r
library(tidyverse)
library(baseballr)
```

I’m going to loop over four and a half MLB seasons and use the
scrape\_statcast\_savant() function to grab the data I need.

``` r
manaea_raw_data <- tribble(~start_date, ~end_date,
                           "2016-01-01", "2016-12-31",
                           "2017-01-01", "2017-12-31",
                           "2018-01-01", "2018-12-31",
                           "2019-01-01", "2019-12-31",
                           "2020-01-01", "2020-09-14") %>%
    # This mutate()/map2() call applies the scrape_statcast_savant() function to every start and end
    # date in the dataframe. The results for each row are put into a separate third column called
    # "data".
    mutate(data=map2(.x=start_date, .y=end_date,
                     ~scrape_statcast_savant(start_date=.x,
                                             end_date=.y,
                                             # This is Manaea's MLBAM player ID, which can be found
                                             # in the URL of any player's page on MLB.com.
                                             playerid=640455,
                                             player_type="pitcher"))) %>%
    # I don't actually need the dates any more, so the next two lines extract the Baseball Savant
    # data and bind them all together into one dataframe.
    pull(data) %>%
    bind_rows()
```

There’s a lot of data here, but I’m going to focus on pitch velocity,
vertical movement, and horizontal movement, found in the
**release\_speed**, **pfx\_x**, and **pfx\_z** columns, respectively. I
also saved the full manaea\_raw\_data dataset generated by the code
block above [in the linked GitHub repository in CSV form
here](https://github.com/dlependorf/manaea_fb_movement/blob/master/manaea_raw_data.csv).

## What did Sean look like before 2020?

A quick pitch movement explainer/refresher: fastballs come out of the
pitcher’s hand with backspin. The more spin a pitcher is able to impart
on a baseball, the more that ball will move during flight. These figures
are relative to a spinless ball, meaning that a pitch with 10 inches of
horizontal movement will move 10 inches leftward during flight relative
to a spinless ball, from the perspective of the TV camera. When dealing
with vertical movement, these figures don’t include the effect of
gravity, so a pitch with 10 inches of vertical movement will “rise” 10
inches more than a spinless ball would. I use “rise” in quotes, because
10 inches of vertical movement isn’t enough to fully counteract how much
a ball falls due to gravity, but it’s enough to make it fall less than
it otherwise would.

With the raw data in tow, what has Manaea’s fastball looked like each
year before 2020?

``` r
old_manaea_fb_only <- manaea_raw_data %>%
    # I'm extracting the season out of the game_date field, and I'm also converting the two
    # pfx_x/pfx_z movement fields from feet to inches.
    mutate(season=as.numeric(format(game_date, "%Y")),
           pfx_x=12*pfx_x,
           pfx_z=12*pfx_z) %>%
    filter(pitch_type=="FF",
           season < 2020)

old_manaea_fb_only %>%
    group_by(season) %>%
    summarize(fastballs=n(),
              release_speed=mean(release_speed),
              pfx_x=mean(pfx_x),
              pfx_z=mean(pfx_z),
              .groups="drop")
```

| season | fastballs | release\_speed | pfx\_x | pfx\_z |
| -----: | --------: | -------------: | -----: | -----: |
|   2016 |      1268 |          92.89 |  17.27 |  12.55 |
|   2017 |      1556 |          91.69 |  14.09 |  10.57 |
|   2018 |      1313 |          90.44 |  14.32 |   9.92 |
|   2019 |       309 |          89.90 |  11.33 |  11.04 |

Sean was pitching through a shoulder impingement in 2017 and 2018 before
having surgery after the 2018 season. This absolutely lines up with how
his fastball behaved: the pitch lost around a tick of mph each year, and
it also lost a few inches of movement in both directions while he was
pitching through pain.

``` r
old_manaea_fb_only_by_season <- old_manaea_fb_only %>%
    group_by(season) %>%
    summarize(release_speed=mean(release_speed),
              pfx_x=mean(pfx_x),
              pfx_z=mean(pfx_z),
              .groups="drop")

ggplot(old_manaea_fb_only_by_season) +
    geom_point(aes(x=pfx_x, y=pfx_z, fill=as.factor(season)),
               shape=21, alpha=0.5, size=30, color="transparent") +
    geom_text(aes(x=pfx_x, y=pfx_z, label=season)) +
    scale_x_continuous("Horizontal Pitch Movement (in)", limits=c(0, 20)) +
    scale_y_continuous("Vertical Pitch Movement (in)", limits=c(0, 20)) +
    coord_equal() +
    labs(title="Sean Manaea Fastball Movement",
         subtitle="By Season, 2016-2019") +
    theme(legend.position="none")
```

![](README_files/figure-gfm/before_2020_graph-1.png)<!-- -->

## Okay, so how about 2020?

After his first four starts in 2020, Sean had yet to complete the 5th
inning, and he was running an ERA of 9.00. Batters were hitting over
.350 against him and slugging almost .600. But since then? Starting with
his appearance on August 5th, Sean has had an ERA under 2, with an
opposing slash line of .204/.228/.286. Bear with me here: it’s not good
analysis to pick what appears to be an inflection point in terms of
performance and draw conclusions on it, but there’s an underlying reason
I used that point to split his season in half. A lot has been made about
a bump in velocity, but aside from the one start on September 5th after
a long COVID-fueled break, there really hasn’t been much in the way of
increased velocity, at least in terms of average fastball velocity
across each start. But what about fastball movement?

``` r
new_manaea_fb_only <- manaea_raw_data %>%
    mutate(season=as.numeric(format(game_date, "%Y")),
           pfx_x=12*pfx_x,
           pfx_z=12*pfx_z) %>%
    filter(pitch_type=="FF",
           season==2020)

new_manaea_fb_only_by_start <- new_manaea_fb_only %>%
    group_by(game_date) %>%
    summarize(fastballs=n(),
              release_speed=mean(release_speed),
              pfx_x=mean(pfx_x),
              pfx_z=mean(pfx_z),
              .groups="drop") %>%
    mutate(start_num=rank(game_date), .before="game_date")
```

| start\_num | game\_date | fastballs | release\_speed | pfx\_x | pfx\_z |
| ---------: | :--------- | --------: | -------------: | -----: | -----: |
|          1 | 2020-07-25 |        24 |          88.98 |   9.75 |  10.95 |
|          2 | 2020-07-31 |        49 |          90.02 |  10.78 |  11.49 |
|          3 | 2020-08-05 |        37 |          90.01 |  10.12 |   9.76 |
|          4 | 2020-08-10 |        35 |          90.93 |  10.11 |  11.83 |
|          5 | 2020-08-15 |        38 |          90.87 |  12.54 |   9.92 |
|          6 | 2020-08-20 |        37 |          89.04 |  13.85 |   8.85 |
|          7 | 2020-08-25 |        46 |          89.90 |  13.72 |  10.98 |
|          8 | 2020-09-05 |        37 |          92.12 |  13.72 |  10.25 |
|          9 | 2020-09-10 |        35 |          90.63 |  15.74 |  11.07 |

That’s a world of a difference, and that dramatic increase in fastball
movement is why I chose to split his season into his first four starts
and then his next five.

``` r
ggplot(new_manaea_fb_only_by_start) +
    geom_point(aes(x=pfx_x, y=pfx_z, color=paste0(start_num, " | ", game_date)),
               shape=1, size=6, stroke=2) +
    geom_text(aes(x=pfx_x, y=pfx_z, label=start_num)) +
    scale_x_continuous("Horizontal Pitch Movement (in)", limits=c(0, 20)) +
    scale_y_continuous("Vertical Pitch Movement (in)", limits=c(0, 20)) +
    coord_equal() +
    guides(color=guide_legend(override.aes=list(size=4))) +
    labs(title="Sean Manaea Fastball Movement",
         subtitle="By 2020 Start, through September 10") +
    theme(legend.title=element_blank())
```

![](README_files/figure-gfm/2020_by_start_graph-1.png)<!-- -->

Sean has pretty drastically increased how much his fastball moves
horizontally over the season, and I don’t think it’s a coincidence that
this increase in movement has gone along with better pitching results.
Adding horizontal movement results in increased “tail”, which is
leftward movement from the perspective of the TV camera for a
left-handed pitcher like Manaea.

With his latest start on September 10th, Manaea has been able to
increase the amount of horizontal movement he’s getting on his fastball
to the point where it’s legitimately close to where it was in 2016.
Overlaying the two plots above makes this change more clear.

``` r
ggplot() +
    geom_point(data=old_manaea_fb_only_by_season,
               aes(x=pfx_x, y=pfx_z, fill=as.factor(season)),
               shape=21, alpha=0.3, size=30, color="transparent") +
    geom_point(data=new_manaea_fb_only_by_start,
               aes(x=pfx_x, y=pfx_z, color=paste0(start_num, " | ", game_date)),
               shape=1, size=6, stroke=2) +
    geom_text(data=new_manaea_fb_only_by_start,
              aes(x=pfx_x, y=pfx_z, label=start_num)) +
    scale_x_continuous("Horizontal Pitch Movement (in)", limits=c(0, 20)) +
    scale_y_continuous("Vertical Pitch Movement (in)", limits=c(0, 20)) +
    coord_equal() +
    guides(color=guide_legend(override.aes=list(size=4)),
           fill=guide_legend(override.aes=list(size=10, alpha=0.5))) +
    labs(title="Sean Manaea Fastball Movement",
         subtitle="By Season 2016-2019, By Start 2020",
         color="2020 Starts",
         fill="2016-2019")
```

![](README_files/figure-gfm/combined_graph-1.png)<!-- -->

I don’t know how Sean will pitch moving forward in 2020, but the
movement on his fastball is absolutely something to keep a close eye on.
This doesn’t mean that velocity isn’t important of course, but there are
other ways to get outs than pure speed: Sean’s path forward from
shoulder surgery may lie in spin and movement instead.
