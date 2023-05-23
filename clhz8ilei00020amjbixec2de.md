---
title: "My F# space adventure"
datePublished: Wed Dec 26 2018 20:20:20 GMT+0000 (Coordinated Universal Time)
cuid: clhz8ilei00020amjbixec2de
slug: my-fsharp-space-adventure
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1684817651945/e18791b7-a17e-47ab-9c9d-ddefe6fda0be.jpeg
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1684783188268/b12a8ed5-062c-416f-b8b6-859db1712005.jpeg
tags: data-science, f, space

---

This post is a part of the [F# Advent Calendar 2018](https://sergeytihon.com/2018/10/22/f-advent-calendar-in-english-2018/).

## Meet Proba 2

![PROBA-2](https://cdn.hashnode.com/res/hashnode/image/upload/v1684567914098/8f45c9a8-e722-4b0d-8a21-e321b8b7f7fa.jpeg align="center")

*Source: ESA,* [*http://www.esa.int/Our\_Activities/Technology/Proba\_Missions/About\_Proba-2*](http://www.esa.int/Our_Activities/Technology/Proba_Missions/About_Proba-2)

[PROBA-2](https://en.wikipedia.org/wiki/PROBA-2) is a small satellite, launched on November 2nd, 2009, and developed under an ESA program in Belgium. PROBA-2 contains five scientific instruments. One of them is the [TPMU](http://www.ufa.cas.cz/html/upperatm/tpmu/TPMU.html) (Thermal Plasma Measurement Unit), developed by the Institute of Atmospheric Physics, Academy of Sciences of the Czech Republic.

My post is about a bug in TPMU and how I used F# to fix it.

Learn more about PROBA-2: [about Proba-2](http://www.esa.int/Our_Activities/Space_Engineering_Technology/Proba_Missions/About_Proba-2)

## History of the bug

When the first data come in, the TPMU team found out there was some issue with the data for electron temperature. It turns out that the wrong program was mistakenly copied to its read-only memory - a testing version that was meant to be used on Earth for tests or experiments. This program is different from the real one because on Earth some sensors don't generate any meaningful data (only some noise) their output was amplified through some electronic components. As a result of this, data from TPMU were malformed in some unknown (but consistent) way.

![PROBA-2 with bug](https://cdn.hashnode.com/res/hashnode/image/upload/v1684733321658/0dea5315-020f-4bc0-8a9a-03e281bda5ac.png align="center")

*Adapted from source: ESA,* [*http://www.esa.int/spaceinimages/Images/2010/01/Proba-2*](http://www.esa.int/spaceinimages/Images/2010/01/Proba-2)

Basically, this can be seen as a bug in production code in read-only memory in space.

## SWARM to the rescue

![SWARM](https://upload.wikimedia.org/wikipedia/en/7/7b/Swarm_spacecraft.jpg align="left")

*Source: ESA, http://spaceinimages.esa.int/Images/2012/03/Swarm*

Luckily, another satellite in space measures the same thing. It's called [SWARM](https://en.wikipedia.org/wiki/Swarm_(spacecraft)). We can compare its data with data from PROBA-2. Let's look at its [histograms](https://en.wikipedia.org/wiki/Histogram):

![PROBA-2 histogram](https://cdn.hashnode.com/res/hashnode/image/upload/v1684733514280/9ba8518c-18bd-4929-8bd1-3073f64169d4.png align="center")

![PROBA-2 histogram](https://cdn.hashnode.com/res/hashnode/image/upload/v1684733553740/1e42f802-beb1-4e3f-a997-dafc00c99b1e.png align="center")

The histogram basically tells us the frequency of numbers among data set.

Can we use this to "fix" our bug in space? We can take SWARM data and treat it as the truth, i.e. how the data should look like, and see, where it leads us. Ultimately, we want to find a function that can be applied to data from TPMU and we get corrected data.

## Percentile to percentile

We can compare data set *A* with data set *B*, by looking at the number on [percentile](https://en.wikipedia.org/wiki/Percentile) *p* in *A* to the number in the same percentile in *B*. Percentile is a number in *p \* size(A)* position of sorted data *A*. The only problem is that the sizes of our data sets are different. That's not hard to solve, we map every number from smaller data set to a number in the corresponding percentile of the bigger data set.

F# code that creates pairs of corresponding (on the same percentile) values follows:

```fsharp
let getPairs xData yData =
    let xs = Array.sort xData
    let ys = Array.sort yData
    let xlen = Array.length xs 
    let ylen = Array.length ys 
    if xlen < ylen then
        xs |> Array.mapi (fun i x -> x, ys.[int <| (float i)*(float ylen)/(float xlen)])
    else
        ys |> Array.mapi (fun i y -> xs.[int <| (float i)*(float xlen)/(float ylen)], y)
```

## The first result

Then we draw a graph from these pairs, where *X* coordinate is our original data and *Y* coordinate is the correct data. The result is very interesting:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1684733904518/0bd481d6-9621-49f2-97c4-4d56b7cfa2ad.png align="center")

We can see that it consists of three blocks that look like [continuous](https://en.wikipedia.org/wiki/Continuous_function) functions, each block is delimited by the visible break in an otherwise smooth graph. There is a good chance that using [polynomial regression](https://en.wikipedia.org/wiki/Polynomial_regression) on these blocks gives us a good transforming function.

## Constructing a fitting function - polynomial regression

The goal of polynomial regression is to find the parameters of the polynomial

$$y : x \mapsto p_0 + p_1 x + p_2 x^2 + \cdots + p_N x^N$$

such that the difference from real data points is minimal.

There is a method for [polynomial regression in Math.NET Numerics](https://numerics.mathdotnet.com/Regression.html#Polynomial-Regression). This method can be used to find optimal parameters

$$p_0, p_1, \cdots, p_N$$

for polynomials of given order from data points.

```fsharp
let fitPoly' order ps =
    let ps = ps |> Seq.toArray
    let xs = ps |> Array.map fst |> Array.map float
    let ys = ps |> Array.map snd |> Array.map float
    let p = Fit.Polynomial(xs, ys, order)
    let f = Fit.PolynomialFunc(xs, ys, order) |> (fun f -> fun x -> f.Invoke(x))
    f, p

/// Get polynomial parameters for fitting function from data pairs ps with polynomial order n
let fitPolyParams n ps = fitPoly' n ps |> snd

/// Get fitting function from data pairs ps with polynomial order n
let fitPoly n ps = 
    let (f, p) = fitPoly' n ps
    fun x ->
        let y = Evaluate.Polynomial((x:float), p)
        assert (f x = y)
        f x
```

## Results for different orders of polynomial

To illustrate how polynomial regression became more and more precise with increasing order, I have created graphs for selected orders:

Polynomial order 3

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1684733934665/0d5d3a60-0666-4e28-9a25-cced3dc1629d.png align="center")

Polynomial order 5

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1684733951157/72461066-c040-4b4d-ac1b-b670e5311959.png align="center")

Polynomial order 10

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1684733963184/c19da3a4-939b-4ac8-821b-7d1e69f4ec9d.png align="center")

Polynomial order 50

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1684733978423/529eba45-16b6-4703-b1be-09072ffce21f.png align="center")

## Final notes

To calculate final values, order 50 is used.

There is one important difference from the described process: data from SWARM were recalculated by clever people at the Institute of Atmospheric Physics to values as if SWARM was at PROBA-2 position (according to the mathematical model).

The results of this work were presented at EGU 2017: [Abstract](https://meetingorganizer.copernicus.org/EGU2017/EGU2017-7152.pdf), [Poster](https://github.com/jindraivanek/blog-hashnode/blob/main/EGU%202017-TPMU-print_version.pdf)

The source code of my tool that is used to correct data from the satellite is available [here](https://gitlab.com/jindraivanek/proba-tool).

### Links:

[http://proba2.sidc.be/index.html/](http://proba2.sidc.be/index.html/)

[http://www.ufa.cas.cz/html/upperatm/tpmu/TPMU.html](http://www.ufa.cas.cz/html/upperatm/tpmu/TPMU.html)

[http://www.esa.int/Our\_Activities/Space\_Engineering\_Technology/Proba\_Missions/About\_Proba-2](http://www.esa.int/Our_Activities/Space_Engineering_Technology/Proba_Missions/About_Proba-2)