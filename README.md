
Sean Manaea has had an up and down 2020, and there has been a lot of
talk around his fastball velocity and how it seems to track closely with
how well he pitches. A pitcher’s velocity is undoubtedly important of
course, but I don’t think the conversation on Manaea has hinged around
the right thing: instead of fastball velocity, we should be talking more
about fastball movement.

I’m going to explain why in this R Markdown document I’m sort of using
as a blog, with all R code embedded within the file if you want to
follow along.

## About Sean Manaea and his shoulder

Sean had shoulder surgery at the end of 2018, and at the time, [Jane Lee
of MLB.com reported
this](https://www.mlb.com/news/a-s-sean-manaea-has-shoulder-surgery-c295372404):

> Manaea, \[A’s head trainer\] Paparesta said, was dealing with chronic
> shoulder impingement for two years, worsening to the point where he
> couldn’t pitch without pain. The 26-year-old lefty was 12-9 with a
> 3.59 ERA in 27 starts this season, twirling a no-hitter against the
> Red Sox on April 21.

In theory, Sean Manaea was healthy in 2016, until his shoulder started
acting up in 2017 and 2018. This means that we might be able to see this
in his fastball movement.

## Grabbing the data

To grab the pitch tracking data needed for this, I’m going to use Bill
Petti’s fantastic [baseballr](https://github.com/BillPetti/baseballr) R
package, which comes bundled with functions that grab raw pitch data
from Baseball Savant. I’m also loading the
[tidyverse](https://www.tidyverse.org/) suite of R packages.

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
                           "2020-01-01", "2020-09-12") %>%
    # This mutate()/map2() call applies the scrape_statcast_savant() function to every start and end
    # date in the dataframe. The results for each row are put into a separate third column called
    # "data".
    mutate(data=map2(.x=start_date, .y=end_date,
                     ~scrape_statcast_savant(start_date=.x,
                                             end_date=.y,
                                             playerid=640455,
                                             player_type="pitcher"))) %>%
    # I don't actually need the dates any more, so the next two lines extract the Baseball Savant
    # data and bind them all together into one dataframe.
    pull(data) %>%
    bind_rows() %>%
    # The only cleanup I'm doing here is converting the pfx_x and pfx_z pitch movement columns from
    # feet to inches.
    mutate(across(pfx_x:pfx_z, ~12*.x))
```

There’s a lot of data here, but I’m going to focus on pitch velocity,
vertical movement, and horizontal movement, found in the
**release\_speed**, **pfx\_x**, and **pfx\_z** columns, respectively.

## What did Sean look like before 2020?

A quick pitch movement explainer/refresher: fastballs come out of the
pitcher’s hand with backspin. The more spin a pitcher is able to impart
on a baseball, the more that ball will move during flight. These figures
are relative to a spinless ball, meaning that a pitch with 10 inches of
vertical movement will “rise” 10 inches relative to a spinless ball. I
use “rise” in quotes, because 10 inches of vertical movement isn’t
enough to fully counteract how much a ball falls due to gravity, but
it’s enough to make it fall less than it otherwise would.

With the raw data in tow, what has Manaea’s fastball looked like each
year before 2020?

``` r
old_manaea_fb_only <- manaea_raw_data %>%
    mutate(season=as.numeric(format(game_date, "%Y"))) %>%
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

Remember that Sean was pitching through a shoulder impingement in 2017
and 2018 before having surgery after 2018. This absolutely lines up with
his fastball, which lost a tick of mph each year, as well as a couple of
inches of movement in both directions while hurt.

``` r
old_manaea_fb_only_by_season <- old_manaea_fb_only %>%
    group_by(season) %>%
    summarize(release_speed=mean(release_speed),
              pfx_x=mean(pfx_x),
              pfx_z=mean(pfx_z),
              .groups="drop")

ggplot(old_manaea_fb_only_by_season) +
    geom_point(aes(x=pfx_x, y=pfx_z, fill=as.factor(season)),
               shape=21, alpha=0.5, size=20, color="transparent") +
    geom_text(aes(x=pfx_x, y=pfx_z, label=season)) +
    scale_x_continuous("Horizontal Pitch Movement (in)", limits=c(0, 20)) +
    scale_y_continuous("Vertical Pitch Movement (in)", limits=c(0, 20)) +
    coord_equal() +
    labs(title="Sean Manaea Fastball Movement",
         subtitle="By Season, 2016-2019") +
    theme(legend.position="none")
```

![](README_files/figure-gfm/before_2020_graph-1.png)<!-- -->

# Okay, so how about 2020?

How has 2020 gone? After his first four starts, Sean had yet to complete
the 5th and had an ERA of 9.00. But since then? Over the five starts
after that, beginning with the one on August 15th, Sean has had an ERA
under 2. At least in terms of results, he’s been pretty dramatically
better. A lot has been made about his bump in velocity, but aside from
the one start after a long COVID-fueled break, there really hasn’t been
very much in the way of increased velocity, at least in terms of average
fastball velocity across each start. But what about fastball movement?

``` r
new_manaea_fb_only <- manaea_raw_data %>%
    mutate(season=as.numeric(format(game_date, "%Y"))) %>%
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

Look at how much his fastball has changed across 2020\!

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
leftward movement for a left-handed pitcher like Manaea from the
perspective of the TV camera.

With his latest start on September 10th, Manaea has been able to
increase the amount of horizontal movement he’s getting on his fastball
to the point where it’s legitimately close to where it was in 2016.
Overlaying the two plots above makes this change more clear.

``` r
ggplot() +
    geom_point(data=old_manaea_fb_only_by_season,
               aes(x=pfx_x, y=pfx_z, fill=as.factor(season)),
               shape=21, alpha=0.3, size=20, color="transparent") +
    geom_point(data=new_manaea_fb_only_by_start,
               aes(x=pfx_x, y=pfx_z, color=paste0(start_num, " | ", game_date)),
               shape=1, size=6, stroke=2) +
    geom_text(data=new_manaea_fb_only_by_start,
              aes(x=pfx_x, y=pfx_z, label=start_num)) +
    scale_x_continuous("Horizontal Pitch Movement (in)", limits=c(0, 20)) +
    scale_y_continuous("Vertical Pitch Movement (in)", limits=c(0, 20)) +
    coord_equal() +
    guides(color=guide_legend(override.aes=list(size=4)),
           fill=guide_legend(override.aes=list(size=8, alpha=0.5))) +
    labs(title="Sean Manaea Fastball Movement",
         subtitle="By Season 2016-2019, By Start 2020",
         color="2020 Starts",
         fill="2016-2019")
```

![](README_files/figure-gfm/combined_graph-1.png)<!-- -->

I don’t know how Sean will pitch moving forward in 2020, but this is
absolutely something to keep watching as Sean continues to pitch.
Velocity is important, to be clear, but the real path forward from
shoulder surgery recovery may lie in spin and movement instead.
