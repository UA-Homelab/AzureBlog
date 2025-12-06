+++
draft = false
title = 'About Me'
layout = 'about'
disableShare = true
+++

<style>
  .about-container {
    display: flex;
    align-items: flex-start;
    gap: 2rem;
  }

  .profile-pic {
    width: 200px;
    flex-shrink: 0;
  }

  @media (max-width: 600px) {
    .profile-pic {
      width: 200px;
      margin-bottom: 0px;
      padding-bottom: 0;
    }

    .about-container {
      flex-direction: column;
      align-items: center;
      margin-top: 0;
      padding-top: 0;
      margin-bottom: 2rem;
    }
  }
</style>

<div class="about-container">
  <img class="profile-pic" src="/me.png" alt="A picture of me">
  <div class="aboutMyWork">
    <br>
    I’m Alexander Urdl, a <span id="age"></span>-year-old tech enthusiast with a passion for solving complex problems, building solutions and see them working (at some point at least).
    <br><br>
    I have <span id="experience"></span> years of experience working in IT. Today, I work as an Azure Solution Architect, specializing in Azure Infrastructure and Infrastructure as Code for the past <span id="cloudExperience"></span> years.
  </div>
</div>

When I’m done working, I spent time with my hobbies:
<ul>
  🎶 Making music<br>
  🏹 Practicing archery<br>
  🌱 Tending to my urban garden<br> 
  🥾 Exploring nature through hiking and biking<br>
  💻 Programming for fun and personal projects<br>
</ul>

<p id="age"></p>
<p id="experience"></p>

<script>
  const birthYear = 1995;
  const startYear = 2016;
  const cloudYear = 2021;
  const currentYear = new Date().getFullYear();

  document.getElementById("age").textContent = `${currentYear - birthYear}`;
  document.getElementById("experience").textContent = `${currentYear - startYear} years`;
  document.getElementById("cloudExperience").textContent = `${currentYear - cloudYear}`;
</script>
