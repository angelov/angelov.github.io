---
layout: post
title:  "Introducing EESTEC Platform for Local Committees"
date:   2014-08-05 21:46:43
categories: 
---

“EESTEC Platform” is a web application which will aim to ease the internal management of EESTEC’s local committees. The main idea for this to be achieved is to automate many of the tasks currently done “by hand” and to collect the big number of information (such as list of members and their info) currently spread across many spreadsheets, papers, mails etc. in one place.

At first, the functionalities of the applications are mainly focused to the board members, but later we should focus more on the regular members and make it possible for them to contribute in the organization’s management like in the “real life”. It’s a fact that the main communication between the members is being done by using Facebook or some other social network and this application won’t aim to change this situation. Also, this application is not intended to be used globally for the whole organization, but every LC would have own separate independent instance of it.

The application is under active development and in its current state it is available as an open source software on GitHub – [https://github.com/angelov/eestec-platform](https://github.com/angelov/eestec-platform), licensed under the GPL-3 license.

The text bellow is divided in three parts: “Application’s functionalities explained”, “Under the hood” and “Contributing to the project”.

Application’s functionalities explained
---

The functionalities are divided into few sections – Dashboard, Members Management, Membership Fees Management and Meetings Management. I will write about each of the sections separately bellow. Some of those functionalities are implemented, some are under development and some of them need to be discussed better. I will also mention a few ideas for other sections planned to be developed in future. The information shown on the included pictures is fake and randomly generated for the presentation’s needs.

###Dashboard

This is a section where mainly are presented many statistical information, nicely represented using different types of charts. For example, there are charts about the number of new members per month in the last year, the number of members per year etc. There’s also a module that lists the members who have a birthday on the current day. The section is easily extensible and the number of ideas can be limitless. Moritz Knüppel had a great idea for a module that would predict the number of members of the LC in the upcoming months based on a few factors.

[<img src="{{ site.url }}/images/eestecplatformintro/dashboard_small.png" class="img-center" style="border:1px solid #999999;" />]({{ site.url }}/images/eestecplatformintro/dashboard.png)

###Members Management

Every member of the LC has their own profile in the application and can be authenticated using their email address and password. There are plans to provide a “social authentication” and make it possible for the members to connect it to their Facebook (or some other social network) profile for easier and quicker access. The members are divided in two categories based on their access level: board members and regular users. The new members can’t create their own profile but they need to be added by some board member instead. In this section, the board members can also list all the members, search by first or last name, preview member’s details etc.

[<img src="{{ site.url }}/images/eestecplatformintro/members_small.png" class="img-center" style="border:1px solid #999999;" />]({{ site.url }}/images/eestecplatformintro/members.png)

[<img src="{{ site.url }}/images/eestecplatformintro/editmember_small.png" class="img-center" style="border:1px solid #999999;" />]({{ site.url }}/images/eestecplatformintro/editmember.png)

###Membership Fees Management

In order to have an active membership, every member has to have the annual membership fee paid. In this section the board members can have an evidence about the paid fees, renew member’s membership by adding new paid fees etc.

[<img src="{{ site.url }}/images/eestecplatformintro/addfee_small.png" class="img-center" style="border:1px solid #999999;" />]({{ site.url }}/images/eestecplatformintro/addfee.png)

###Meetings Management

The weekly meetings are an important part of the organization’s working. Beside the main info about the date and the location, every meeting has its own agenda and list of attendants. In this section, the board members can create a report after every meeting which will include information from said meeting. Adding the members as attendants in the report is made to be very easy by providing an auto-complete option to suggest members’ names. The information from these reports can later be used for many things, like calculating a member’s attendance rate. This can be really useful later to generate a list of members who will have a right to vote on the upcoming annual local congress where the new board members are being elected.

[<img src="{{ site.url }}/images/eestecplatformintro/meeting_small.png" class="img-center" style="border:1px solid #999999;" />]({{ site.url }}/images/eestecplatformintro/meeting.png)

The development of a few sections of the application that are planned to be included too, is projected to be started in the near future. Bellow you can read about a few of the main ideas.

###Events Management

Over the year, there are many workshops, exchanges, motivational weekends etc. organized by the many other LCs. Lots of members are interested for those events, many of them apply and some of them get accepted. As Angela Spirkoska said, it will be very useful for the board members to have a list of the events and can see who of the local members have applied for them and see their acceptance status. For this section to work properly, it has to be connected to the main EESTEC site, in order to automatically get the information when a new event is published. There will be a better discussion with the EESTEC’s IT team about the needed API.

###Address Book

The main source for financing the work of the LC and their organized events are the local companies and institutions. There are teams of members whose job is to contact the companies when, for example, a new event is being organized and request a sponsorship. And as they say, every company is a separate experience. This section will contain a list of these companies and their contact info. The members would be able to share their experience with each of them. There can be some kind of a suggestion system which will be able to suggest which companies to contact next.

###Office Availability Management

I don’t know about the other LCs, but LC Skopje’s office is available for the members whenever they need it for some organizational work. To avoid overlapping and having too many people in the office, we need a way to have the members informed when will the office be available so they can request to use it. Viktorija Nikolovska’s idea is to implement some sort of a calendar which will show the usage of the office. This will be great both for the board and the regular members. The board members will have a better evidence of who has and will use the office and for what reasons, and everyone will have an easy way to see when they can plan their activities there.

Under the hood
---

To write about how the things are made to be working, I will separate this part of the text in two sub-parts: Back-end and Front-end.

###Back-end

The application is based on the Laravel PHP framework. The architecture of its back-end is structured in a few separate layers: Controllers (for handling the HTTP requests and returning responses), Services, Domain layer (the Model namespace), Data Persistence layer (the Repository namespace) and Presentation layer (the views). The responses can either contain compiled HTML templates or just JSON encoded data, depending on the type of the request. To separate the presentation from the business logic, we’re using the Twig template engine (instead of the Laravel’s default engine called Blade). To handle the data persistence, there are few repositories which currently use the Eloquent ORM. They can be easily changed if there’s need because the application relies on the abstraction, not their concrete implementation. The code aims to follow the PSR-2 coding standard. Currently there aren’t written any tests for the code, so if have knowledge about PHPUnit, Behat, Codeception or something similar, your help is more than welcome.

###Front-end

The design of the application is currently being developed using the great Bootstrap CSS framework. For the JavaScript part, the application uses the jQuery framework to manipulate some parts of the DOM and send AJAX requests to the back-end. This currently works nicely, however, there is a possibility that it could be changed to use the AngularJS framework.

Contributing to the project
---

Everything about this project is public and everyone is welcome to contribute. The project can get big and in order to make it useful sooner, we need anyone’s help. There aren’t any application forms you need to fill to apply before contributing, any interviews or anything similar. It doesn’t matter if you are a programmer, part of some LC’s board or just a regular member of the organization. Anyone can contribute anytime. If you want to contribute to the source code, feel free to submit a pull request to the project’s repository on GitHub and your code will be reviewed. If you find any bugs you can report them by opening a new issue in the same repository. You can also use the issues to suggest new functionalities for the system or to discuss the existing ones.

At the end, I would like to thank some of the members of LC Skopje and the EESTEC’s IT team for the help they provided.

If you have any questions about the project or you want to try the application but you can’t install it, feel free to contact me on my email address [angelovdejan92 at gmail.com] or on [Facebook](https://facebook.com/angelovdejan).