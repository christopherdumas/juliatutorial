# 12 - DataFrames

## Dataframes

Julia has a library to handle tabular data, in a way similar to R or Pandas dataframes. The name is, no surprises, [DataFrames](https://github.com/JuliaStats/DataFrames.jl). The approach and the function names are similar, although the way of actually accessing the API may be a bit different.  
For complex analysis, [DataFramesMeta](https://github.com/JuliaStats/DataFramesMeta.jl) adds some helper macros.

## Documentation:

* DataFrames: [https://dataframesjl.readthedocs.io/en/latest/getting\_started.html](https://dataframesjl.readthedocs.io/en/latest/getting_started.html), [http://juliastats.github.io/DataFrames.jl/](http://juliastats.github.io/DataFrames.jl/), [https://juliastats.github.io/DataFrames.jl/stable/man/reshaping\_and\_pivoting/](https://juliastats.github.io/DataFrames.jl/stable/man/reshaping_and_pivoting/), [https://en.wikibooks.org/wiki/Introducing\_Julia/DataFrames](https://en.wikibooks.org/wiki/Introducing_Julia/DataFrames)
* DataFramesMeta: [https://github.com/JuliaStats/DataFramesMeta.jl](https://github.com/JuliaStats/DataFramesMeta.jl)
* Stats in Julia in general: [http://juliastats.github.io/](http://juliastats.github.io/)

### Install and import the library

* Install the library: `Pkg.add(DataFrames)`
* Load the library: `using DataArrays, DataFrames`

### Create a df or load data:

* From a table:

```text
using CSV
supplytable = CSV.read(IOBuffer("""
prod Epinal Bordeaux Grenoble
Fuelwood 400 700 800
Sawnwood 800 1600 1800
Pannels 200 300 300
"""))
```

* Read a CSV file: `myData = CSV.read(file; delim=';', null="\N", delim=";", decimal=',')` \(use `CSV.read(file; delim=' ')` for comma separated values\)

  If a column has in the first top rows used by type-autorecognition only missing values, but then has non-missing values in subsequent rows, an error may appear. The trick is to manually specify the column value with the `type` parameter \(Vector or Dictionary, e.g. `type=Dict("freeDim" => Union{Missing,Int64})`\)

* From a stream, use the package `Request`:

  ```text
  using DataFrames, Requests
  resp = get("https://data.cityofnewyork.us/api/views/kku6-nxdu/rows.csv?accessType=DOWNLOAD")
  df = readtable(IOBuffer(resp.data))
  ```

* Crate a df from scratch:

  ```text
  df = DataFrame(
  colour = ["green","blue","white","green","green"],
  shape  = ["circle", "triangle", "square","square","circle"],
  border = ["dotted", "line", "line", "line", "dotted"],
  area   = [1.1, 2.3, 3.1, 4.2, 5.2])
  ```

* Create an empty df: `df = DataFrame(A = Int64[], B = Float64[])`

If a column if found to have all NA values, it will be treated by default as a Int64. In this case use `convert()` if you want to store other type of values: `df[:col] = convert(DataArrays.DataArray{Float64,1}, df[:col])`

* Convert from a Matrix of data and a vector of column names: `convert(DataFrame, Dict(zip(headerstrs,[mat[:,i] for i in 1:size(mat,2)])))`
* Convert from a Matrix with headers in the first row: `convert(DataFrame,Dict(zip(mat[1,:],[mat[2:end,i] for i in 1:size(mat,2)])))`

### Get insights about your data:

* `head(df)`
* `showall(df)`
* `tail(df)`
* `describe(df)`
* `unique(df[:fieldName])` or `[unique(df[i]) for i in names(df)]`
* `count(df[:fieldName])`
* `names(df)` returns array of column names
* `size(df)` \(r,c\), `size(df)[1]` \(r\), `size(df)[2]` \(c\)
* `ENV["LINES"] = 60` change the default number of lines before the content is truncated \(default 30\). Also COLUMNS. May not work with terminal.
* `for r in eachrow(df)` iterates over each row

Column names are Julia symbols. To programmatically compose a column name you need hence to use the Symbol\(String\) constructor, e.g.:  
`df[Symbol("value_"*string(0))] = "aa"`

### Edit data

* Replace values based to a dictionary : `mydf[:col1] = map(akey->myDict[akey], mydf[:col1])` \(the original data to replace can be in a different column or a totally different dataframe
* Concatenate \(string\) values for several columns to create the value a new column: `df[:c] = df[:a] .* " " .* df[:b]`

  \(in julia pre-0.6, due to a bug, use instead `df[:c] = df[:a] * " " .* df[:b]`\)

* To compute the value of a column based of other columns you need to use  elementwise operations using the dot, e.g. `df[:a] = df[:b] .* df[:c]` \(note that the equal sign doesn't have the dot.. but if you have to make a comparation `==` operator wants also the dot, i.e. `.==`\)
* Append a row: `push!(df, [1 2 3])`
* Delete a given row: use `deleterows!()` or just copy a df without the rows that are not needed, e.g. `df2 = df[[1:(i-1);(i+1):end],:]` 
* Empty a dataframe: `df = similar(df,0)`

#### Filter

* Filter by value, based on a field being in a list of values: `df[indexin(df[:colour], ["blue","green"]) .> 0, :]`
* Alternative using list comprehension: `df[ [i in ["blue","green"] for i in df[:colour]], :]`
* Combined boolean selection: `df[(indexin(df[:colour], ["blue","green"]) .> 0) .& (df[:shape] .== "triangle"), :]` \(the dot is needed to vectorize the operation\). Note the usage of the bitwise and \(single ampersand\).
* Filter using `@where` \(`DataFrameMeta` package\): `@where(df, :x .> 2, :y .== "a")  # the two expressions are "and-ed"`. If the column name is stored in a variable, you need to wrap it using the `_I_()` function, e.g. `col = Symbol("x");  @where(df, _I_(col) .> 2)`
* Change a single value by filtering columns: `df[ (df[:product] .== "hardWSawnW") .& (df[:year] .== 2010) , :consumption] = 200`
* Filter based on initial pattern: `filteredDf = df[startswith.(df[:field],pattern),:]`
* A benchmark note: using `@with()` or boolean selection is ~ the same, while "querying" an equivalent Dict with categorical variables as tuple keys is around ~20% faster than querying the dataframe.

### Edit structure

* Delete columns by name: `delete!(df, [:col1, :col2])`
* Rename columns: `names!(df, [:c1,:c2,:c3])` \(all\) `rename!(df, Dict(:c1 => :neCol))` \(a selection\)
* Change column order: `df = df[[:b, :a]]`
* Add an "id" column \(useful for unstacking\): `df[:id] = 1:size(df, 1)`  \# this makes it easier to unstack
* Add a Float64 column \(all filled with NA by default\): `df[:a] = DataArray(Float64,size(df,1))`
* Add a column based on values of other columns: `df[:c] =  df[:a]+df[:b]` \(alternative: use map: `df[:c] = map((x,y) -> x + y, df[:a], df[:b])`\)
* Insert a column at a given position: \(1\) save the name before adding the new column `dfnames = names(df)`; \(2\) add the column; \(3\) `df = df[vcat(dfnames[1:1],:newCol,dfnames[2:end])]`
* Convert columns:
  * from Int to Float: `df[:A] = convert(DataArrays.DataArray{Float64,1},df[:A])`
  * from Float to Int: `df[:A] = convert(DataArrays.DataArray{Int64,1},df[:A])`
  * from Int \(or Float\) to String: `df[:A] = map(string, df[:A])`
  * from String to Float: `string_to_float(str) = try parse(Float64, str) catch return(NA) end; df[:A] = map(string_to_float, df[:A])`
  * from DataArray{T} to Array{T\]: `a = Array(df[:a])`
  * from Any to T \(including String, if the individual elements are already strings\): `df[:A] = convert(DataArrays.DataArray{T,1},df[:A])`
* You can "pool" specific columns in order to efficiently store repeated categorical variabels with `pool!(df, [:A, :B])`. Attenction that while the memory decrease, filtering with pooled values is not quicked \(indeed it is a bit slower\)

#### Merge/Join/Copy datasets

* Concatenate different dataframes \(with same structure\): ```df = vcat(df1,df2,df3)`` or `df = vcat([df1,df2,df3]...)` \(note the three dots at the end\).
* Join dataframes orizzontally: `fullDf = join(df1, df2, on = :commonCol)`
* Copy the structure of a DataFrame \(to an empty one\): `df2 = similar(df1, 0)`

### OLD::Manage NA values

* The NA value is simply `NA`
* `complete_cases!(df)` or `complete_cases(df)` select only rows without NA values \(you can specify on which columns you want to apply this filter with `complete_cases!(df[[:col1,:col2]])` or `df2 = df[complete_cases(df[[:col1]]),:]`\)
* Within an operation \(e.g. a sum\) you can use `dropna()` in order to skip NA values before the operation take place.
* `[df[isna.(df[i]), i] = 0 for i in names(df)]` remove NA values on all columns
* To use filtering \(either boolean filtering or `@where` macro in `DataFramesMeta`\) where NA values could be present or may be requested esplicitly use `isequal.(a,b)` as otherwise the confrontation \(`==`\) with NA values leads to NA values and not the expected boolean ones.
* Count the `NA` values: `nNA = length(find(x -> isna(x), df[:col]))`

### Manage Missing values \(NEW\)

Starting from DataFrames 0.11.3, NA has been replaced with missing object \(Missing type\) from package Missings.jl In Julia &gt;= 0.7 Missings will be in core \(with some additional functionality still provided by the additional package Missings.jl\). A DataFrame change from being a collection of DataArrays to a colelction of simple Arrays, eventually of type Union{T,Missing} if missing data is present.

* The missing value is simply `missing`
* Remove missing values with: `a = collect(Missings.skipmissings(df[:col1]))` \(returns an Array\) or `b = dropmissing(df[[:col1,:col2]])` \(return a DataFrame even for a single column\) 
* `complete_cases!(df)` or `complete_cases(df)` select only rows without NA values \(you can specify on which columns you want to apply this filter with `complete_cases!(df[[:col1,:col2]])` or `df2 = df[complete_cases(df[[:col1]]),:]`\)
* Within an operation \(e.g. a sum\) you can use `dropmissing()` in order to skip NA values before the operation take place.
* `[df[isna.(df[i]), i] = 0 for i in names(df)]` remove NA values on all columns
* To use filtering \(either boolean filtering or `@where` macro in `DataFramesMeta`\) where NA values could be present or may be requested esplicitly use `isequal.(a,b)` as otherwise the confrontation \(`==`\) with NA values leads to NA values and not the expected boolean ones.
* Count the `missing` values: `nMissings = length(find(x -> ismissing(x), df[:col]))`

### Split-Apply-Combine strategy

The DataFrames package supports the Split-Apply-Combine strategy through the `by` function, which takes in three arguments: \(1\) a DataFrame, \(2\) a column \(or columns\) to split the DataFrame on, and \(3\) a function or expression to apply to each subset of the DataFrame.

The function can return a value, a vector, or a DataFrame. For a value or vector, these are merged into a column along with the `cols` keys. For  
a DataFrame, `cols` are combined along columns with the resulting DataFrame. Returning a DataFrame is the clearest because it allows column labeling.

Alternativly the `by` function can take the function as first argument, so to allow the usage of do blocks.  
Inside, it uses the groupby\(\) function, as in the code it is defined as nothing else than:

```text
by(d::AbstractDataFrame, cols, f::Function) = combine(map(f, groupby(d, cols)))
by(f::Function, d::AbstractDataFrame, cols) = by(d, cols, f)
```

#### Aggregate/

Aggregate by several fields:

* `aggregate(df, [:field1, :field2], sum)`

  Attenction that all categorical fields have to be included in the list of fields on which to aggregate, otherwise julia will try to comput a sum over them \(that being string will rice an error\) instead of just ignoring them.

  The workaround is to remove the fields you don't want before doing the operation.

* Alternativly:

  ```text
  by(df, [:catfield1,:catfield2]) do df
    DataFrame(m = sum(df[:valueField]))
  end
  ```

#### Compute cumulative sum by categories

* Manual method \(very slow\):

  ```text
  df = DataFrame(region=["US","US","US","US","EU","EU","EU","EU"],
               year = [2010,2011,2012,2013,2010,2011,2012,2013],
               value=[3,3,2,2,2,2,1,1])
  df[:cumValue] = copy(df[:value])
  [r[:cumValue] = df[(df[:region] .== r[:region]) .& (df[:year] .== (r[:year]-1)),:cumValue][1] + r[:value]  for r in eachrow(df) if r[:year] != minimum(df[:year])]
  ```

* Using by and the split-apply-combine strategy \(fast\):

  ```text
  using DataFramesMeta, DataArrays, DataFrames
  df = DataFrame(region  = ["US","US","US","US","EU","EU","EU","EU"],
               product = ["apple","apple","banana","banana","apple","apple","banana","banana"],
               year    = [2010,2011,2010,2011,2010,2011,2010,2011],
               value   = [3,3,2,2,2,2,1,1])
  df[:cumValue] = copy(df[:value])
  by(df, [:region,:product]) do dd
    dd[:cumValue] = cumsum(dd[:value])
       return
  end
  ```

* Using @linq \(from DataFramesMeta\) and the split-apply-combine strategy \(fast\):

  ```text
  using DataFramesMeta, DataArrays, DataFrames
  df = DataFrame(region  = ["US","US","US","US","EU","EU","EU","EU"],
               product = ["apple","apple","banana","banana","apple","apple","banana","banana"],
               year    = [2010,2011,2010,2011,2010,2011,2010,2011],
               value   = [3,3,2,2,2,2,1,1])
  df = @linq df |>
  groupby([:region,:product]) |>
  transform(cumValue = cumsum(:value))
  ```

* Using groupby \(fast\):

  ```text
  using DataFramesMeta, DataArrays, DataFrames
  df = DataFrame(region  = ["US","US","US","US","EU","EU","EU","EU"],
               product = ["apple","apple","banana","banana","apple","apple","banana","banana"],
               year    = [2010,2011,2010,2011,2010,2011,2010,2011],
               value   = [3,3,2,2,2,2,1,1])
  df[:cumValue] = 0.0
  for subdf in groupby(df,[:region,:product])
    subdf[:cumValue] = cumsum(subdf[:value])
  end
  ```

### Pivot

#### Stack

Move columns to rows of a "variable" column, i.re. moving from wide to long format.  
For `stack(df,[cols])` you have to specify the column\(s\) that have to be stacked, for `melt(df,[cols])` at the opposite you specify the other columns, that represent the id columns that are already in stacked form.  
Finally `stack(df)` - without column names - automatically stack all float columns.  
Note that the stacked columns are inserted as row of a "variable" column \(with names of the variables not strings but symbols\) and the corresponding values in a "column" value.

```text
df = DataFrame(region = ["US","US","US","US","EU","EU","EU","EU"],
               product = ["apple","apple","banana","banana","apple","apple","banana","banana"],
               year = [2010,2011,2010,2011,2010,2011,2010,2011],
               produced = [3.3,3.2,2.3,2.1,2.7,2.8,1.5,1.3],
               consumed = [4.3,7.4,2.5,9.8,3.2,4.3,6.5,3.0])
long1 = stack(df,[:produced,:consumed])
long2 = melt(df,[:region,:product,:year])
long3 = stack(df)
long1 == long2 == long3 # true
```

#### Unstack

You can specify the dataframe, the column name which content will become the row index \(id variable\), the column name with content will become the name of the columns \(column variable names\) and the column name containing the values that will be placed in the new table \(column values\):

`widedf = unstack(longdf, :id, :variable, :value)`

Alternativly you can oit the :id parameter and all the existing column except the one defining column names and the one defining column values will be preserved as index \(row\) variables:

`widedf = unstack(longdf, :variable, :value)`

#### Sorting

`sort!(df, cols = (:col1, :col2), rev = (false, false))` The \(optional\) reverse order parameter \(rev\) must be a turple of the same size as the cols parameter

### Export your data

writetable\("file.csv", df, separator = ';', header = false\)

See also the section [Interfacing Julia with other languages](../language-core/interfacing-julia-with-other-languages.md) to get an example on how to import/export data from an ods file.

#### Export to Dict \#1

This export to a dictionary where the keys are the unique elements of a df column and the values are the splitted dataframes:

```text
vars = Dict{String,DataFrame}()
[vars[x] = @where(data, :varName .== x) for x in unique(dataata[:varName])]
[delete!(vars[k], [:varName]) for k in keys(vars)]
```

#### Use hdf5 format

sudo apt-get install hdf5-tools Pkg.add\("HDF5"\)
