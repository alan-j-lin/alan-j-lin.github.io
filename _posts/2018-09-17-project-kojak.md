---
layout: post
title: Improving Youtube’s Demonetization Algorithm for Sex Educators
---

According to an article in the Journal of Adolescent Health by Hall et. al titled [The State of Sex Education in the United States] there are currently less than 15 states that require sex education be medically accurate. At the same, they observed that while the quality of sex education in the United States went down from 2006 to 2014, there was an analogous trend where teen pregnancies went down and the use of contraceptions for teenagers went up during the same time period. They stated that:

 ```

 These coincident trends suggest that adolescents are receiving information about birth control and condoms elsewhere.

 ```

By elsewhere, I think a natural interpretation would mean the Internet. In particular, I believe platforms like Youtube provide access to important sex education related content. Some content creators of particular import in this space are Lindsay Doe (a clinical sexologist that runs the [Sexplanations] channel), [Hannah Witton], and [Laci Green].

![](/public/Project_Kojak/sexplanations_vid1.png)

![](/public/Project_Kojak/hannahwitton_vid1.png)

However while such creators provide valuable information for public consumption, many of these videos are blocked from earning ad-revenue automatically upon upload to Youtube due to their supposedly "explicit nature". Because most views from subscribers generally occur within the first day of upload, this greatly affects these creators' revenue streams even if the creators are able to successfully appeal for their videos to be remonetized. So the central question is whether or not an alternate approach to Youtube's algorithm could be developed so that it doesn't affect these content creators disproportionately? For my fifth and final project at [Metis] I created an audio and text based model to accomplish this.

[The State of Sex Education in the United States]: www.article.com
[Sexplanations]: www.youtube.com/sexplanations
[Hannah Witton]: www.youtube.com/hannahwitton
[Laci Green]: www.youtube.com/lacigreen
[Metis]: https://www.thisismetis.com/

## Data

### Acquiring Data:

In Youtube's reasoning to demonetize these videos due to their explicit nature, I decided that my model had to be able to distinguish between the educational content and truly explicit content, i.e. pornography. To do so I obtained 125 sex education-related videos from Youtube and 125 videos from Pornhub, where both types of videos were from the most popular categories in their respective spaces. These videos were then cut down to 5000+ 30 second video segments which totaled to roughly 42 hours worth of video. These 5000+ segments were then what my model was going to classify as either explicit or educational.

### Pipeline:
