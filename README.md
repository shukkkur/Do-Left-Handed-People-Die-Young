![Forks](https://img.shields.io/github/forks/shukkkur/Do-Left-Handed-People-Die-Young.svg)
![Stars](https://img.shields.io/github/stars/shukkkur/Do-Left-Handed-People-Die-Young.svg)
![Watchers](https://img.shields.io/github/watchers/shukkkur/Do-Left-Handed-People-Die-Young.svg)
![Last Commit](https://img.shields.io/github/last-commit/shukkkur/Do-Left-Handed-People-Die-Young.svg) 

<h2 align='center'>Do Left-handed People Really Die Young?</h2>


<p>A 1991 study reported that left-handed people die on average nine years earlier than right-handed people.<br>Let's explore this phenomenon using age distribution data to see if we can reproduce a difference in average age at death purely from the changing rates of left-handedness over time, refuting the claim of early death for left-handers. </p>

<h3>Datasets</h3>

- [Death distribution data](https://www.cdc.gov/nchs/data/statab/vs00199_table310.pdf) for the United States from the year 1999
- [Rates of left-handedness](https://pubmed.ncbi.nlm.nih.gov/1528408/)

<h3>1. Where are the old left-handed people?</h3>

<p>A National Geographic survey in 1986 resulted in over a million responses that included age, sex, and hand preference for throwing and writing. Researchers Avery Gilbert and Charles Wysocki analyzed this data and noticed that rates of left-handedness were around 13% for people younger than 40 but decreased with age to about 5% by the age of 80. They concluded based on analysis of a subgroup of people who throw left-handed but write right-handed that this age-dependence was primarily due to changing social acceptability of left-handedness. <br>Let's start by plotting the rates of left-handedness as a function of age.</p>

```python
data_url_1 = "https://gist.githubusercontent.com/mbonsma/8da0990b71ba9a09f7de395574e54df1/raw/aec88b30af87fad8d45da7e774223f91dad09e88/lh_data.csv"
lefthanded_data = pd.read_csv(data_url_1)

fig, ax = plt.subplots() # create figure and axis objects
ax.plot(lefthanded_data.Age, lefthanded_data.Female,label='Female', marker = 'o') # plot "Female" vs. "Age"
ax.plot(lefthanded_data.Age, lefthanded_data.Male, label='Male', marker = 'x') # plot "Male" vs. "Age"

plt.show()
```

<img src='datasets/img1.jpg'>

<h3>2. Rates of left-handedness over time</h3>

<p>Let's convert this data into a plot of the rates of left-handedness as a function of the year of birth, and average over male and female to get a single rate for both sexes.<br>Since the study was done in 1986, the data after this conversion will be the percentage of people alive in 1986 who are left-handed as a function of the year they were born.</p>

```python
lefthanded_data['Birth_year'] = 1986 - lefthanded_data.Age
lefthanded_data['Mean_lh'] = (lefthanded_data.Male+lefthanded_data.Female)/2

fig, ax = plt.subplots()
ax.plot(lefthanded_data.Birth_year, lefthanded_data.Mean_lh) # plot 'Mean_lh' vs. 'Birth_year'

plt.show()
```
<img src='datasets/img2.jpg'>

<h3>3. Applying Bayes' rule</h3>

<p>We want to calculate the <b>probability of dying at age A given that you're left-handed</b>.<br>Here's Bayes' theorem for the two events we care about: left-handedness <code>LH</code> and dying at age <code>A</code>.</p>
<p align='center'>
  <img src='datasets/formula1.jpg'>
</p>

<ul>
  <li><code>P(LH | A)</code> the probability that you are left-handed given that you died at age <code>A</code></li>
  <li><code>P(A)</code> overall probability of dying at age <code>A</code></li>
  <li><code>P(LH)</code> overall probability of being left-handed</li>
</ul>

<p>To calculate <code>P(LH | A)</code> for ages that might fall outside the original data, we will need to extrapolate the data to earlier and later years.</p>

```python
def P_lh_given_A(ages_of_death, study_year = 1990):
    """ P(Left-handed | age of death), calculated based on the reported rates of left-handedness.
    Inputs: age of death, study_year
    Returns: probability of left-handedness given that a subject died in `study_year` at age `age_of_death` """
    
    # Use the mean of the 10 neighbouring points for rates before and after the start 
    early_1900s_rate = lefthanded_data['Mean_lh'][-10:].mean()
    late_1900s_rate = lefthanded_data['Mean_lh'][:10].mean()
    middle_rates = lefthanded_data.loc[lefthanded_data['Birth_year'].isin(study_year - ages_of_death)]['Mean_lh']
    
    youngest_age = study_year - 1986 + 10 # the youngest age in the NatGeo dataset is 10
    oldest_age = study_year - 1986 + 86 # the oldest age in the NatGeo dataset is 86
    
    P_return = np.zeros(ages_of_death.shape)  # create an empty array to store the results
    # extract rate of left-handedness for people of age age_of_death
    P_return[ages_of_death > oldest_age] = early_1900s_rate / 100
    P_return[ages_of_death < youngest_age] = late_1900s_rate / 100
    P_return[np.logical_and((ages_of_death <= oldest_age), (ages_of_death >= youngest_age))] = middle_rates / 100
 
    return P_return
```

<h3>4. When do people normally die? <code>P(A)</code></h3>
<p>To estimate the probability of living to an age <code>A</code>, we can use data that gives the number of people who died in a given year and how old they were to create a distribution of ages of death.</p>

```python
# Death distribution data for the United States in 1999
data_url_2 = "https://gist.githubusercontent.com/mbonsma/2f4076aab6820ca1807f4e29f75f18ec/raw/62f3ec07514c7e31f5979beeca86f19991540796/cdc_vs00199_table310.tsv"
death_distribution_data = pd.read_csv(data_url_2, sep='\t', skiprows=[1])

death_distribution_data = death_distribution_data.dropna(subset=['Both Sexes'])

fig, ax = plt.subplots()
ax.plot(death_distribution_data['Age'], death_distribution_data['Both Sexes']) # plot 'Both Sexes' vs. 'Age'

plt.show()
```

<img src='datasets/img33.jpg'>

<h3>5. The overall probability of left-handedness <code>P(LH)</code></h3>
<p><code>P(LH)</code> is the probability that a person who died in our particular study year is left-handed. We can calculate it by summing up all of the left-handedness probabilities for each age, weighted with the number of deceased people at each age, then divided by the total number of deceased people to get a probability.</p>
<p align='center'>
  <img src='datasets/formula.jpg'>
</p>

<ul>
  <li><code>N(A)</code> is the number of people who died at age A</li>
</ul>

```python
def P_lh(death_distribution_data, study_year = 1990): # sum over P_lh for each age group
    """ Overall probability of being left-handed if you died in the study year
    P_lh = P(LH | Age of death) P(Age of death) + P(LH | not A) P(not A) = sum over ages 
    Input: dataframe of death distribution data
    Output: P(LH), a single floating point number """
    p_list = death_distribution_data['Both Sexes']*P_lh_given_A(death_distribution_data['Age'], study_year)
    p = np.sum(p_list)
    return p/np.sum(death_distribution_data['Both Sexes']) # normalize to total number of people in distribution

>>> P_lh(death_distribution_data)
... 0.07766387615350638
```

<h3>6. Putting it all together: dying while left-handed (i)</h3>
<p>Now we have the means of calculating all three quantities we need: <code>P(A)</code>, <code>P(LH)</code> and <code>P(LH | A)</code><br>We can combine all three using Bayes' rule to get <code>P(A | LH)</code><br>.</p>

<p>For left-handers, </p>

```python
def P_A_given_lh(ages_of_death, death_distribution_data, study_year = 1990):
    """ The overall probability of being a particular `age_of_death` given that you're left-handed """
    P_A = death_distribution_data['Both Sexes'][ages_of_death] / np.sum(death_distribution_data['Both Sexes'])
    P_left = P_lh(death_distribution_data, study_year) # use P_lh function to get probability of left-handedness overall
    P_lh_A = P_lh_given_A(ages_of_death, study_year) # use P_lh_given_A to get probability of left-handedness for a certain age
    return P_lh_A*P_A/P_left
```
<p>And for right-handers, </p>

```python
def P_A_given_rh(ages_of_death, death_distribution_data, study_year = 1990):
    """ The overall probability of being a particular `age_of_death` given that you're right-handed """
    P_A = death_distribution_data['Both Sexes'][ages_of_death] / np.sum(death_distribution_data['Both Sexes'])
    P_right = 1- P_lh(death_distribution_data, study_year) # either you're left-handed or right-handed, so these sum to 1
    P_rh_A = 1-P_lh_given_A(ages_of_death, study_year) # these also sum to 1 
    return P_rh_A*P_A/P_right
```

<h3>8. Plotting the distributions of conditional probabilities</h3>
<p>Let's plot these probabilities for a range of ages of death from 6 to 120.</p>

```python
ages = np.arange(6,115,1)

left_handed_probability = P_A_given_lh(ages, death_distribution_data)
right_handed_probability = P_A_given_rh(ages, death_distribution_data)

fig, ax = plt.subplots() # create figure and axis objects
ax.plot(ages, left_handed_probability, label = "Left-handed")
ax.plot(ages, right_handed_probability, label = "Right-handed")

plt.show()
```
<img src='datasets/img4.jpg'>

<p>Notice that the left-handed distribution has a bump below age 70: of the pool of deceased people, left-handed people are more likely to be younger. </p>

<h3>9. Moment of truth: age of left and right-handers at death</h3>
<p>Finally, let's compare our results with the original study that found that left-handed people were nine years younger at death on average. We can do this by calculating the mean of these probability distributions in the same way we calculated <code>P(LH)</code> earlier, weighting the probability distribution by age and summing over the result.</p>

<img src='datasets/formula4.jpg'>

```python
average_lh_age =  np.nansum(ages*np.array(left_handed_probability))
average_rh_age =  np.nansum(ages*np.array(right_handed_probability))


>>> print(round(average_lh_age,1))
... 67.2
>>> print(round(average_rh_age,1))
... 72.8
>>> print("The difference in average ages is " + str(round(average_rh_age - average_lh_age, 1)) + " years.")
... The difference in average ages is 5.5 years.
```
<h3>10. Final comments</h3>
<p>We got a pretty big age gap between left-handed and right-handed people purely as a result of the changing rates of left-handedness in the population.<br>To finish off, let's calculate the age gap we'd expect if we did the study in 2021 instead of in 1990.</p>

```python
left_handed_probability_2021 = P_A_given_lh(ages, death_distribution_data, study_year = 2021)
right_handed_probability_2021 = P_A_given_rh(ages, death_distribution_data, study_year = 2021)
    
average_lh_age_2021 =  np.nansum(ages*np.array(left_handed_probability_2021))
average_rh_age_2021 =  np.nansum(ages*np.array(right_handed_probability_2021))

>>> print("The difference in average ages is " + str(round(average_rh_age_2021 - average_lh_age_2021, 1)) + " years.")
... The difference in average ages is 1.9 years.
```
