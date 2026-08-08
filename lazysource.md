# Delay-loaded series

The `DelayedSeries` type provides an efficient way to create series whose data is loaded
on-demand. For example, you may have a large time series stored in a CSV file or in a
database and you do not want to load all the data in memory if the user only needs a
small part of it.

When you create a delayed series, you specify the overall range of the series (i.e. the
minimum and maximum key value) and you provide a function that loads a specified sub-range
of the series. When the user accesses a continuous range of the series, the loading function
is called to retrieve the data.

<a name="create"></a>
## Creating a delayed series

To create a delayed series, we need a function that generates data for a given range.
The following function generates a series with random data for a given date range with
a day frequency:

```fsharp
let generate (low:DateTime) (high:DateTime) : seq<KeyValuePair<DateTime,float>> = 
    let rnd = Random()
    let days = int (high - low).TotalDays
    seq [ for d in 0 .. days -> KeyValuePair(low.AddDays(float d), rnd.NextDouble()) ]
```

Now we use `DelayedSeries.FromValueLoader` to create a delayed series. It takes the overall
minimum and maximum key of the series and a function that loads data for a sub-range. The
loading function gets the lower and upper bound as a tuple of `(key, BoundaryBehavior)`
values where `BoundaryBehavior` is either `Inclusive` or `Exclusive`:

```fsharp
let min = DateTime(2010, 1, 1)
let max = DateTime(2013, 1, 1)

let ls = DelayedSeries.FromValueLoader(min, max, fun (lo, lob) (hi, hib) -> async {
    printfn "Query: %A - %A" lo hi
    let lo = if lob = BoundaryBehavior.Inclusive then lo else lo.AddDays(1.0)
    let hi = if hib = BoundaryBehavior.Inclusive then hi else hi.AddDays(-1.0)
    return generate lo hi })
```

The key thing about the above is that, so far, no data has been loaded. The loading function
is called only when we access part of the series.

<a name="slicing"></a>
## Slicing and using delayed series

We can now use the series as usual - for example, to get data for the entire year 2012:

```fsharp
let slice = ls.[DateTime(2012, 1, 1) .. DateTime(2012, 12, 31)]
slice
```

```
val slice: Series<DateTime,float> =
  
(Delayed series [01/01/2012 .. 12/31/2012]) 

val it: Series<DateTime,float> =
  
(Delayed series [01/01/2012 .. 12/31/2012])
```

Similarly, we can add the delayed series to a data frame. When doing this, Deedle will
only load the data that is needed. In the following example, we add the series to a frame
and then access only a slice:

```fsharp
let df = frame ["Values" => ls]
let slicedDf = df.Rows.[DateTime(2012,6,1) .. DateTime(2012,6,30)]
slicedDf
```

```
Query: 01/01/2010 00:00:00 - 01/01/2013 00:00:00
Query: 06/01/2012 00:00:00 - 06/30/2012 00:00:00
val df: Frame<DateTime,string> =
  
              Values              
01/01/2010 -> 0.25313210592817526 
01/02/2010 -> 0.7746178769759903  
01/03/2010 -> 0.49983862287050806 
01/04/2010 -> 0.6735173171911598  
01/05/2010 -> 0.3281452035209118  
01/06/2010 -> 0.7060584543541705  
01/07/2010 -> 0.880403496209692   
01/08/2010 -> 0.22651800016112456 
01/09/2010 -> 0.176219704099992   
01/10/2010 -> 0.9264592464685278  
01/11/2010 -> 0.674440706820305   
01/12/2010 -> 0.21468717528129289 
01/13/2010 -> 0.07282548832183455 
01/14/2010 -> 0.4123421180605832  
01/15/2010 -> 0.3801321490329207  
:             ...                 
12/18/2012 -> 0.8788558520690178  
12/19/2012 -> 0.6525187234471864  
12/20/2012 -> 0.23010524223900397 
12/21/2012 -> 0.12011710443317758 
12/22/2012 -> 0.4429383441274186  
12/23/2012 -> 0.5254668224027516  
12/24/2012 -> 0.6678116035102096  
12/25/2012 -> 0.12042260648249359 
12/26/2012 -> 0.1171408956085146  
12/27/2012 -> 0.878848287635863   
12/28/2012 -> 0.8134949856179191  
12/29/2012 -> 0.3287133256199123  
12/30/2012 -> 0.5609944254800079  
12/31/2012 -> 0.38484024145100026 
01/01/2013 -> 0.7240823465533394  

val slicedDf: Frame<DateTime,string> =
  
              Values              
06/01/2012 -> 0.16797751113563597 
06/02/2012 -> 0.49343826393784995 
06/03/2012 -> 0.1790843998271634  
06/04/2012 -> 0.7419053202821645  
06/05/2012 -> 0.17615172752514374 
06/06/2012 -> 0.9193546144010704  
06/07/2012 -> 0.8116584045385453  
06/08/2012 -> 0.7127041026678838  
06/09/2012 -> 0.0444433875442477  
06/10/2012 -> 0.659710177756751   
06/11/2012 -> 0.63465988018621    
06/12/2012 -> 0.8176484841128501  
06/13/2012 -> 0.39670139096308665 
06/14/2012 -> 0.8232305577960254  
06/15/2012 -> 0.08867466762881748 
06/16/2012 -> 0.47513165404414026 
06/17/2012 -> 0.40351336519228065 
06/18/2012 -> 0.8148471263595817  
06/19/2012 -> 0.856131934664584   
06/20/2012 -> 0.4280442046669346  
06/21/2012 -> 0.668732997024974   
06/22/2012 -> 0.41747271112130635 
06/23/2012 -> 0.2806568775744879  
06/24/2012 -> 0.8276296402861466  
06/25/2012 -> 0.7538178010816372  
06/26/2012 -> 0.17243368817349491 
06/27/2012 -> 0.7469468077100471  
06/28/2012 -> 0.3858544245830684  
06/29/2012 -> 0.6413568207804714  
06/30/2012 -> 0.9556236559283195  

val it: Frame<DateTime,string> =
  
              Values              
06/01/2012 -> 0.16797751113563597 
06/02/2012 -> 0.49343826393784995 
06/03/2012 -> 0.1790843998271634  
06/04/2012 -> 0.7419053202821645  
06/05/2012 -> 0.17615172752514374 
06/06/2012 -> 0.9193546144010704  
06/07/2012 -> 0.8116584045385453  
06/08/2012 -> 0.7127041026678838  
06/09/2012 -> 0.0444433875442477  
06/10/2012 -> 0.659710177756751   
06/11/2012 -> 0.63465988018621    
06/12/2012 -> 0.8176484841128501  
06/13/2012 -> 0.39670139096308665 
06/14/2012 -> 0.8232305577960254  
06/15/2012 -> 0.08867466762881748 
06/16/2012 -> 0.47513165404414026 
06/17/2012 -> 0.40351336519228065 
06/18/2012 -> 0.8148471263595817  
06/19/2012 -> 0.856131934664584   
06/20/2012 -> 0.4280442046669346  
06/21/2012 -> 0.668732997024974   
06/22/2012 -> 0.41747271112130635 
06/23/2012 -> 0.2806568775744879  
06/24/2012 -> 0.8276296402861466  
06/25/2012 -> 0.7538178010816372  
06/26/2012 -> 0.17243368817349491 
06/27/2012 -> 0.7469468077100471  
06/28/2012 -> 0.3858544245830684  
06/29/2012 -> 0.6413568207804714  
06/30/2012 -> 0.9556236559283195
```
