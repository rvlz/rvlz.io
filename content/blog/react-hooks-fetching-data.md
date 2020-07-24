---
title: "Fetching Data with React Hooks"
date: 2020-06-16T12:20:05-05:00
draft: false
tags:
  - "react"
  - "react hooks"
  - "axios"
---

In this tutorial, we'll cover how to use React Hooks to fetch data from the [Breaking Bad API](https://breakingbadapi.com).

Before we examine any JSX code let's take a look at how the data that we're fetching is structured.
All fields reference primitives (*string* and *number*) with the exception of "occupation" and "appearance",
which refer to array objects. (Some fields were omitted for the sake of brevity; if you'd like to see the complete
JSON object, visit the API's [documentation page](https://breakingbadapi.com/documentation).)

```json
{
  "char_id": 1,
  "name": "Walter White",
  "birthday": "09-07-1958",
  "occupation": [
      "High School Chemistry Teacher",
      "Meth King Pin"
  ],
  "appearance": [1, 2, 3, 4, 5]
}
```

Using this information, we build a component to present the data retrieved from the API.

```jsx
import React, { useState } from 'react';

function BreakingBadCharacters() {
  const [characters, setCharacters] = useState([]);

  return (
    <React.Fragment>
      <h1>Break Bad Characters</h1>
      <ul>
        {characters.map(({name, birtday, appearance, occupation}) => (
          <li key={name}>
            <p>{name} was born on {birthday}.</p>
            <p>occupation(s): {occupation.join(', ')}</p>
            <p>season appearance(s): {appearance.join(', ')}</p>
          </li>
        ))}
      </ul>
    </React.Fragment>
  );
}

export default BreakingBadCharacters;
```

When this functional component mounts, we want it to fetch data from the API and update its state with this data.
To do this, we need to include the *useEffect* hook in our component and pass it a function that will do the fetching
and update the component's state.

In this demonstration, we'll use a 3rd party dependency called
axios to communicate with the API. Feel free to use any HTTP client of your choosing.

```jsx
import React, { useState, useEffect } from 'react'; // new
import axios from 'axios'; // new

const API_URL = 'https://www.breakingbadapi.com/api/characters?limit=10'; // new

function BreakingBadCharacters() {
  const [characters, setCharacters] = useState([]);

  // new
  useEffect(() => {
    const fetchBreakingBadData = async () => {
      const response = await axios.get(API_URL);
      setCharacters(response.data);
    };
    fetchBreakingBadData();
  }, []);

  return (
    <React.Fragment>
      <h1>Breaking Bad Characters</h1>
      <ul>
        {characters.map(({name, birthday, appearance, occupation}) => (
          <li key={name}>
            <p>{name} was born on {birthday}.</p>
            <p>occupation(s): {occupation.join(', ')}</p>
            <p>season appearance(s): {appearance.join(', ')}</p>
          </li>
        ))}
      </ul>
    </React.Fragment>
  );
}

export default BreakingBadCharacters;
```

If you look closely, there's an additional argument passed to *useEffect*: an empty array. Since *useEffect*
gets called every time a component mounts or a component's props or state update, *fetchBreakingBadData*&mdash;which updates
*BreakingBadCharacters*'s state&mdash;will be called over and over again. To avoid an infinite loop, we pass an empty array, effectively
telling React to call the passed function only when the component mounts.

After a successful response returns, *fetchBreakingBadData* adds the API data to the component's state, allowing the component to present
the retrieved Breaking Bad character information.

To learn more about using React Hooks visit the official [documentation page](https://reactjs.org/docs/hooks-intro.html).

Thanks for reading!
