---
title: "Types of Technical Interviews"
date: 2023-04-08T13:44:22+02:00
draft: false
slug: "interview-types"
categories: ["Tech"]
tags: ["eng", "interview", "tech"]
cover:
  image: /img/posts/interview-types/cover.jpeg
---

Initially, I wrote [this article in Ukrainian](https://dou.ua/forums/topic/42749/), but I decided to translate it to English and share it with a broader audience. I also added some suggestions from the comments and removed some regional-specific references, but overall the text is mostly unchanged compared to the original.

Why am I even writing this? I participate in the hiring process for my company. Also, I've gone through quite a few interviews in my life. Many folks are looking for a job, and the market is not particularly welcoming. Thus, I want to share some insights on the hiring process that could help you to prepare for the interviewing process and know what to expect at each stage. A couple of disclaimers first:

- This article comes from my personal experience. There might be outliers to this process. My goal here is to cover a generic interview framework.
- My position is Site Reliability Engineer. Thus, my interviewing experience is limited to this type of role. Yet, I believe the process is more or less the same for other positions in software engineering as well.
- I'm not the best person to advise you on how to get your first job. I got mine about a decade ago. Things were very different back then. And I feel like the requirements for entry-level jobs have increased drastically.
- I only worked in product companies throughout my career. The hiring process for outsourcing or consultancy companies may have specifics unknown to me.
- I live in Germany. Therefore, my experience is specific to Western Europe. So, your mileage may vary.
- I won't cover the topic of my CV. There are a ton of good articles written about it. You can watch this [video from Google](https://youtu.be/BYUy1yvjHxE) about how to write a CV. It is very nice. Also, you can find an example of my CV [here](https://grem1.in/cv/).

With this being said, let's get to the topic.

## Types of the Interviews

I distinguish five types of interviews: Introduction, Technical Skills Evaluation, Big Picture Interview, Behavioral (Cultural) Interview, and Closure.

{{< figure src="/img/posts/interview-types/cover.jpeg" alt="cover image" class="center" height="640" >}}

It's important to mention that interview types do not equal interview stages. Sometimes, an interview of a specific type may be omitted or mixed with a different type of interview. For example, Big Picture Interviews may be the case for Junior roles. Also, an interviewer may ask behavioral questions at each stage. However, in this article, I will sometimes use the term "stage" in the meaning of "type."

{{< figure alt="9 stages in the interview process" src="/img/posts/interview-types/screenshot.jpeg" class="center" height="800" >}}

There are nine (!) stages of interviews in the screenshot above. Yet, as you can see, each corresponds to one of the mentioned types. So, let's focus on each type now.

## Introduction

Also known as Screening. This type of interview always comes first. It includes cold messages on LinkedIn and first calls with a recruiter or a hiring manager. At this stage, a company introduces itself to the candidate and learns more about them. From my experience, some Dutch companies and consultancy firms may give IQ tests (1) at this point. I don't know what's the point of it. However, if you want to work in the Netherlands or a company like PWC or Deloitte, you must accept it.

**What is this type for?** To know each other. Here, the company and the candidate "sell" themselves to one another. Also, based on the information you provide recruiter can shape the next steps of the interview. For example, a recruiter can pass this information down to the team if you have a lot of experience with a given technology. In the meantime, you can learn basic information about the company.

**How to prepare**. It may seem like one doesn't have to prepare for this type of interview. Yet, this is your chance to ask some crucial questions early before a lot of time is invested into the process on both sides. For example, the ability to work from home is
essential to me. Hence, I ask about it straight away. If a company requires a full-time presence in the office, it likely doesn't make sense to us to continue the process.

Another important question that usually appears in this stage is compensation. There are various thoughts on when to disclose their desired salary and who should say the number first. I will get back to it at the end of this article.

## Technical Skills Evaluation

The name says it all. This type exists to evaluate a person as a specialist. This type also includes all sorts of live coding interviews and home assessments. Let me elaborate a bit more on the latter.

I know that coding challenges are controversial. I prefer live coding over home tasks, but such interviews may be too stressful for some folks. Ideally, a company allows one to choose between live coding and a home task if the coding challenge is a must.

Also, live coding and home assessment cover different skills. Live coding allows the interviewer to understand the thought process of a person. A home assessment helps to evaluate how one works with technical requirements and final documentation. Although, the value of coding challenges decreases with the level of a specialist (2).

**How to prepare**. There is no universal advice here. Revisit nuances of your domain. Try to recall some typical questions from your previous interviews. If you told the recruiter that you are a specialist with a specific technology, revisit some theory of this technology.

You can also ask the recruiter directly what to expect at this stage and what information you need to recall. Recruiters are often very open about the process and may help here or even provide some books and other materials for preparation.

A couple more words on home assessments. If you have agreed to do it, please, read the requirements carefully! My company gives home challenges to candidates. You will be surprised how often we get back the code that doesn't work or the code that covers some bonus tasks but not some required ones. Needless to say that we do not move forward with such candidates. These challenges exist not because we need more REST APIs but to check how one can work with the requirements and provide a usable output of their work.

Tell the recruiter right away. In case you understand that you cannot complete the challenge in time, be transparent about it. It will leave a good impression on you. So you will end on a good note.

## Big Picture Interview

At first, I wanted to call this type "Systems Design" because this is what interviews I conduct in my company. However, this type covers other things like, for example, case studies. A company can present its case or ask about a situation from your experience. While the importance of coding challenges decreases with the seniority of a candidate, the value of Big Picture interviews only increases, in my opinion.

**What is this type for**. Here the company evaluates that the candidate knows not only _how_ to complete a task but also _why_. The company can check how the candidate works with other people and teams, if they know the problem of local maximums (3), etc. Yet, this is still a technical interview, so you must emphasize the technical side.

**How to prepare**. There is a [excellent book](https://www.amazon.com/Designing-Data-Intensive-Applications-Reliable-Maintainable/dp/1449373321) on systems design. Also, check out the resources from the [Awesome Systems Design List](https://github.com/madd86/awesome-system-design). Be mindful of non-functional requirements! It is the juice of system design interviews. Usually, you face the task of building a simple system. For example, a service to upload cat pictures. Yet, how many photos are expected? How many users are there? Are these users in the same location or not? This interview aims to evaluate if the candidate understands how different systems interact and the tradeoffs of each solution.

For the case studies, prepare a couple of cases of complex or non-standard tasks from your experience. Use [the STAR Method](https://www.themuse.com/advice/star-interview-method) - **S**ituation, **T**ask, **A**ction, **R**esult. 
Example:

> We have identified that some pages take a long to open (situation). We examined the logs and metrics (task) and identified too many requests for the same information in the database. We have added the caching layer (action). Thus, we managed to decrease the latency by X times in the 95th percentile (result). New situations may occur in the action phase. It's up to you whether you want to mention these situations.

## Behavioral Interview

Also known as culture fit. Usually, these are case studies with the hiring manager. Sometimes technical specialists neglect this stage. However, this type is very important!

- Previous interviews could be done by people from other teams or even external consultants. This might be your first time speaking to someone you will work with.
- Every company has a salary range. Often behavioral interviews exist to calibrate the candidate in this range. The salary range may be €75k/Y - €85k/Y. Thus, your final offer could vary for €10k/Y, which is considerable money.

**How to prepare**. All the case study tips from the previous paragraph work here as well. However, it's important to remember to emphasize teamwork and communication skills this time. Prepare a couple of cases when you had to show your leadership skills or align the work between multiple teams. The STAR method and storytelling tips work here as well. 

Example:

> We noticed that some pages take a long to load because of non-optimized DB queries (situation). We checked it with the respective team (task), but they had different priorities. So, we decided to increase the instance size of the database (tradeoff). At the same time, we agreed with that team that they fix their service in the next quarter. Once they did that, we scaled back the database to the previous size (result).

Here is a [great video](https://youtu.be/hU6BVxtGd5g) on "A Life Engineered" channel about this type of interview. Also, he has another [great video](https://youtu.be/0Z9RW_hhUT4) about technical interviews, where he emphasizes the big picture and cultural fit. I highly recommend watching both videos!

## Closure

This is when all the steps are behind, and you're negotiating the final offer. It may seem like this stage has nothing to do with the interviewing process, but it's not. First, you can swap places with the company and do a reverse interview here. You can learn more about reverse interviews on [Gergely Orosz's blog](https://blog.pragmaticengineer.com/reverse-interviewing/). He also has a list of [12 questions one should ask a company](https://blog.pragmaticengineer.com/pragmatic-engineer-test/). Of course, you usually have time to ask questions at each stage. If you're satisfied with the answers, you can skip this phase. Also, some questions from the list are specific to the location. For instance, equity could be a considerable part of the total compensation in the US, but it's just another bonus in Europe.

Since we're already talking money, let's get back to the salary question. There is general advice to discuss the compensation at a very late stage. This strategy is suggested in the "[Ten Rules for Negotiating a Job Offer](https://haseebq.com/my-ten-rules-for-negotiating-a-job-offer/)" article and its [second part](https://haseebq.com/how-not-to-bomb-your-offer-negotiation/). This is not always the best strategy, though.

For example, I always say the number during the initial call. I am aware of my position in the market. I know the approximate salary ranges for my location. I don't want to downgrade my compensation for a similar company. Therefore, saying the number first allows me to cut out companies that won't meet my expectations in the later stages. Although I still leave some room to maneuver. I do not say out loud my desired compensation but my current one. Given that I'm not actively looking for a job, a company has to come up with an attractive offer, and then we can negotiate more.

Otherwise, negotiating your salary later is the best strategy for most people unless you want to filter companies you interview for and not wise versa. Also, remember that monetary compensation is not the whole package. One company may have a bigger self-development budget, while another has more vacation days. Make sure to evaluate all the perks you get when making a decision.

You can check the market situation on the websites such as [Glassdoor](https://www.glassdoor.com/index.htm), [Levels.fyi](https://www.levels.fyi/), [Comprehensive.io](https://www.comprehensive.io/), and others. Yet, be aware of the bias of these websites. From my experience, numbers on Glassdoor are understated, while the <Levels.fyi> situation is the opposite. Also, you can go out and do a couple of interviews yourself. This method won't provide a large data sample but will show how much companies are willing to pay _you_, not some abstract Senior Software Engineer.

That's all, folks! I wish you the best of luck with your interviews. If you have any questions or suggestions, feel free to contact me! You can find my contacts in the "About" section of the website or Substack, depending on where you are reading this piece.

---

1. I use the term "IQ test" here to generalize all personality and situational thinking tests.
2. Here's a good article on live coding interviews - "[Why Senior Engineers Hate Coding Interviews](https://medium.com/swlh/why-senior-engineers-hate-coding-interviews-d583d2855757)."
3. Let's say you have a team T evaluating two technologies to complete their task. Technology A fits their requirements and is easier to maintain. However, technology B allows other teams to solve problems because it has more features. If team T is unaware of the latter, they will likely choose technology A, which suits them but not the whole company. Thus, technology A is a local maximum for team T. This problem is described in Tanya Reilly's "[The Staff Engineer's Path](https://www.amazon.com/Staff-Engineers-Path-Tanya-Reilly-ebook/dp/B0BG16Y553) book.
