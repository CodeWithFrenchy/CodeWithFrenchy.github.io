---
title: Gestion des dates
date: 2026-01-01 19:00:00 -0400
categories: [à mettre]
tags: [à mettre]
---

private (DateOnly DateCourante, DateTime DateHeureCouranteUtc) ObtenirDatesCourantes()
{
    DateTimeOffset dateHeureCouranteUtc = _timeProvider.GetUtcNow();

    DateTimeOffset dateHeureCouranteLocale = TimeZoneInfo.ConvertTime(dateHeureCouranteUtc, _timeProvider.LocalTimeZone);

    return (DateOnly.FromDateTime(dateHeureCouranteLocale.DateTime), dateHeureCouranteUtc.UtcDateTime);
}
