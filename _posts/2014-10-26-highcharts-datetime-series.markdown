---
layout: post
title:  "Highchart datetime axis formatting"
date:   2014-10-08 21:15:20
categories: web highcharts
---
<script src="http://ajax.googleapis.com/ajax/libs/jquery/1.8.2/jquery.min.js"></script>
<script src="http://momentjs.com/downloads/moment.js"></script>
<script src="http://code.highcharts.com/highcharts.js"></script>

If you have worked with [Highcharts][highcharts], you know that it's sometimes difficult to find all the right incantations to get it to do your biding.

One that particularly stumped me was the datetime axis type. I had regular data, for example some metric per month, with my points having an X value of the first day of each month. At first, everything seems fine:


{% highlight javascript %}
$(function () {
  $('#container').highcharts({
    title: {
      text: 'Everything seems fine',
    },
    xAxis: {
      type: 'datetime'
    },
    series: [{
      name: 'A fine series',
      data: [
        [moment('2014-01-01').valueOf(), 7.0],
        [moment('2014-02-01').valueOf(), 6.9],
        [moment('2014-03-01').valueOf(), 9.5],
        [moment('2014-04-01').valueOf(), 14.5],
        [moment('2014-05-01').valueOf(), 18.2],
        [moment('2014-06-01').valueOf(), 21.5],
        [moment('2014-07-01').valueOf(), 25.2],
        [moment('2014-08-01').valueOf(), 26.5],
        [moment('2014-09-01').valueOf(), 23.3],
        [moment('2014-10-01').valueOf(), 18.3],
        [moment('2014-11-01').valueOf(), 13.9],
        [moment('2014-12-01').valueOf(), 9.6]
      ]
    }]
  });
});
{% endhighlight %}

<div id="container-fine" style="min-width: 310px; height: 400px; margin: 0 auto"></div>

<script type="text/javascript">
$(function () {
  $('#container-fine').highcharts({
    title: {
      text: 'Everything seems fine',
    },
    xAxis: {
      type: 'datetime',
    },
    series: [{
      name: 'See? Fine.',
      data: [
        [moment('2014-01-01').valueOf(), 7.0],
        [moment('2014-02-01').valueOf(), 6.9],
        [moment('2014-03-01').valueOf(), 9.5],
        [moment('2014-04-01').valueOf(), 14.5],
        [moment('2014-05-01').valueOf(), 18.2],
        [moment('2014-06-01').valueOf(), 21.5],
        [moment('2014-07-01').valueOf(), 25.2],
        [moment('2014-08-01').valueOf(), 26.5],
        [moment('2014-09-01').valueOf(), 23.3],
        [moment('2014-10-01').valueOf(), 18.3],
        [moment('2014-11-01').valueOf(), 13.9],
        [moment('2014-12-01').valueOf(), 9.6]
      ]
    }]
  });
});
</script>

The problem
-----------

So far, so good. But then, you add a date format for the labels, because your client doesn't like Highcharts' default format:

{% highlight javascript %}
    // ...
    xAxis: {
      // ...
      labels: {
        formatter: function() {
          return moment(this.value).format("YYYY-MM-DD");
        }
      }
    },
    // ...
{% endhighlight %}

<div id="container-offbyone" style="min-width: 310px; height: 400px; margin: 0 auto"></div>

<script type="text/javascript">
$(function () {
  $('#container-offbyone').highcharts({
    title: {
      text: 'My Labels!',
    },
    xAxis: {
      type: 'datetime',
      labels: {
        formatter: function() {
          return moment(this.value).format("YYYY-MM");
        }
      }
    },
    series: [{
      name: 'What happened to my labels?!?',
      data: [
        [moment('2014-01-01').valueOf(), 7.0],
        [moment('2014-02-01').valueOf(), 6.9],
        [moment('2014-03-01').valueOf(), 9.5],
        [moment('2014-04-01').valueOf(), 14.5],
        [moment('2014-05-01').valueOf(), 18.2],
        [moment('2014-06-01').valueOf(), 21.5],
        [moment('2014-07-01').valueOf(), 25.2],
        [moment('2014-08-01').valueOf(), 26.5],
        [moment('2014-09-01').valueOf(), 23.3],
        [moment('2014-10-01').valueOf(), 18.3],
        [moment('2014-11-01').valueOf(), 13.9],
        [moment('2014-12-01').valueOf(), 9.6]
      ]
    }]
  });
});
</script>

Furthermore, your client complains that sometimes the labels repeat themselves when he generates the report:

<div id="container-repeatingdates" style="min-width: 310px; height: 400px; margin: 0 auto"></div>

<script type="text/javascript">
$(function () {
  $('#container-repeatingdates').highcharts({
    title: {
      text: 'What?',
    },
    xAxis: {
      type: 'datetime',
      labels: {
        formatter: function() {
          return moment(this.value).format("YYYY-MM");
        }
      }
    },
    series: [{
      name: 'That\'s impossible!',
      data: [
        [moment('2014-01-01').valueOf(), 7.0],
        [moment('2014-02-01').valueOf(), 6.9],
        [moment('2014-03-01').valueOf(), 9.5],

        /*
        */
      ]
    }]
  });
});
</script>

Explanation
-----------

Well, as i've found, there are actually two different issues:

1. There is an offset between the point on the chart and the tick on the axis (as evidenced by the first chart).
2. The labels repeat themselves because Highcharts falls on a smaller scales (days, not months), and your date format hides this from you.

Problem #1
----------

Let me illustrate point 1 with a more obvious example:

<div id="container-offbyone-worse" style="min-width: 310px; height: 400px; margin: 0 auto"></div>

<script type="text/javascript">
$(function () {
  $('#container-offbyone-worse').highcharts({
    title: {
      text: 'Issue #1',
    },
    xAxis: {
      type: 'datetime',
      labels: {
        formatter: function() {
          return moment(this.value).format("YYYY-MM-DD HH:mm:ss");
        }
      }
    },
    series: [{
      name: 'It should be obvious',
      data: [
        [moment('2014-01-01').valueOf(), 7.0],
        [moment('2014-01-02').valueOf(), 6.9],
        [moment('2014-01-03').valueOf(), 9.5],
        [moment('2014-01-04').valueOf(), 14.5],
        [moment('2014-01-05').valueOf(), 18.2],
        [moment('2014-01-06').valueOf(), 21.5],
        [moment('2014-01-07').valueOf(), 25.2],
        [moment('2014-01-08').valueOf(), 26.5],
        [moment('2014-01-09').valueOf(), 23.3],
        [moment('2014-01-10').valueOf(), 18.3],
        [moment('2014-01-11').valueOf(), 13.9],
        [moment('2014-01-12').valueOf(), 9.6]
      ]
    }]
  });
});
</script>

You may not be seeing the same thing as me, but I get labels offset by -4 hours, which look suspiciously like a UTC vs localized dates issue. After trying a few solutions mostly involving Moment.js wizardry (amazing library, by the way), i stumbled across [this][highcharts-useUTC-doc]. The solution is quite simple:

{% highlight javascript %}
// Add this before rendering your chart
Highcharts.setOptions({
  global: {
    useUTC: false
  }
});
{% endhighlight %}

<div id="container-offbyone-fixed" style="min-width: 310px; height: 400px; margin: 0 auto"></div>

<script type="text/javascript">
$(function () {
  Highcharts.setOptions({
    global: {
      useUTC: false
    }
  });
  $('#container-offbyone-fixed').highcharts({
    title: {
      text: 'Issue #1, fixed',
    },
    xAxis: {
      type: 'datetime',
      labels: {
        formatter: function() {
          return moment(this.value).format("YYYY-MM-DD HH:mm:ss");
        }
      }
    },
    series: [{
      name: 'Yay!',
      data: [
        [moment('2014-01-01').valueOf(), 7.0],
        [moment('2014-01-02').valueOf(), 6.9],
        [moment('2014-01-03').valueOf(), 9.5],
        [moment('2014-01-04').valueOf(), 14.5],
        [moment('2014-01-05').valueOf(), 18.2],
        [moment('2014-01-06').valueOf(), 21.5],
        [moment('2014-01-07').valueOf(), 25.2],
        [moment('2014-01-08').valueOf(), 26.5],
        [moment('2014-01-09').valueOf(), 23.3],
        [moment('2014-01-10').valueOf(), 18.3],
        [moment('2014-01-11').valueOf(), 13.9],
        [moment('2014-01-12').valueOf(), 9.6]
      ]
    }]
  });
});
</script>

Problem #2
----------

Now, on to issue #2. As before, let's have an example to better illustrate the issue:

<div id="container-repeatingdates-days" style="min-width: 310px; height: 400px; margin: 0 auto"></div>

<script type="text/javascript">
$(function () {
  $('#container-repeatingdates-days').highcharts({
    title: {
      text: 'Issue #2',
    },
    xAxis: {
      type: 'datetime',
      labels: {
        formatter: function() {
          return moment(this.value).format("YYYY-MM-DD");
        }
      }
    },
    series: [{
      name: 'Those are not months',
      data: [
        [moment('2014-01-01').valueOf(), 7.0],
        [moment('2014-02-01').valueOf(), 6.9],
        [moment('2014-03-01').valueOf(), 9.5]
      ]
    }]
  });
});
</script>

This one stumped me a lot longer than issue #1. I tried various possible solutions, including defining the axis as a 'category' axis (don't go there, you may not come back), but I finally found the solution, again in the [Highcharts documentation][highcharts-minTickInterval]:

{% highlight javascript %}
    // ...
    xAxis: {
      // ...
      minTickInterval: moment.duration(1, 'month').asMiliseconds()
      //                                   ^^^^^ or whatever your interval is
    },
    // ...
{% endhighlight %}

And voilà, the beautiful charts you've always wanted:

<div id="container-repeatingdates-fixed" style="min-width: 310px; height: 400px; margin: 0 auto"></div>

<script type="text/javascript">
$(function () {
  $('#container-repeatingdates-fixed').highcharts({
    title: {
      text: 'Issue #2, fixed',
    },
    xAxis: {
      type: 'datetime',
      minTickInterval: moment.duration(1, 'month').asMilliseconds(),
      labels: {
        formatter: function() {
          return moment(this.value).format("YYYY-MM");
        }
      }
    },
    series: [{
      name: 'Those ARE months',
      data: [
        [moment('2014-01-01').valueOf(), 7.0],
        [moment('2014-02-01').valueOf(), 6.9],
        [moment('2014-03-01').valueOf(), 9.5]
      ]
    }]
  });
});
</script>

Conclusion
----------

Highcharts is a very powerful library, but it can be hard sometimes to get it to do exactly what you want. However, as we've demonstrated here, there's usually a way. Feel free to tweet me any question.

[highcharts]:                 http://www.highcharts.com/
[highcharts-useUTC-doc]:      http://api.highcharts.com/highcharts#global.useUTC
[highcharts-minTickInterval]: http://api.highcharts.com/highcharts#xAxis.minTickInterval
