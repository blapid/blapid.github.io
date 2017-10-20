---
layout: post
title:  "LLCPP - A quest to faster C++ logging (Part 2)"
date:   2017-10-19 12:56:15 +0300
categories: cpp
---

Several weeks ago a friend and I were discussing C++ logging frameworks. He had some issues with his company's home-made logger, mainly regarding call latencies, and wanted to switch to something better. After some googling around we've found a couple of interesting candidates: [g3log][g3log-github], [NanoLog][NanoLog-github], [MAL][mal-github] and [spdlog][spdlog-github]. All four options aim to be "fast" and use similar approaches to achieve that. In this post I'd like to talk about what logging frameworks do, how do they define and measure what "fast" is and showcase a different approach to C++ logging which I've been messing around with.
