## What is react query?

- A library for data fetching
- great for handling server state
    - server state is persisted remotely and requires async APIs for fetching and updating 
- helps in caching, performance optimizations, updating stale data in background 

## Installing react query 

- for v3.19.2, at the top of component tree in App.js, we need to add QueryClientProvider 
- you also need to create queryclient instance
- we pass this instance as a prop to QueryClientProvider

## Fetching data using useQuery hook 

useQuery hook requires two args
1) unique key that could be a str to identify query 
2) function that returns a promise, for fetching it will have the get req to the server endpt 

useQuery returns both entire data and isLoading var 

we don't have to manage state flags for loading, error as well as error thrown in request 

## DevTools for React Query 

```
<ReactQueryDevtools initialIsOpen={false} position='bottom-right'/>
```

- Typically devtools display the queries on the page along with the query key 
-  Clicking on a query gives more details and provide UI for foll actions 
    - refetch, invalidate, reset, remove 

- After actions, we have data explorer, similar to Network tab of browser dev tools 
- Also query explorer provides config details about query (not imp)

## queryCache

- By default every query result is cached for 5mins in React query 
- useQuery provides another bool flag for background fetching of query ```isFetching```
- You can change default value of 5mins, by passing another argument to useQuery
    - an object with key: {cacheTime: 5000}

## Stale Time 

- Another use of queryCache is to reduce no of network requests for data that doesn't change too often 
- To achive this, we configure another property called staleTime
```{cacheTime: 50000, staleTime: 30000}```
    - staleTime of 30s and cacheTime of 50s
- default staleTime is 0s, i.e whenever we revisit the page, it would reftech data from server 

## refetchOnMount 

- by default it is true 
- refetches on mount if data is stale 
- possible values true, false, always
-  when always is selected, it will refetch the data when component mounts 

## refetchOnWindowFocus

- by default it is true 
- so any time your tab loses and gains focus, a background fetch is initiated 
- helps in syncing UI and remote data from server
- possible values true, false, always

## Polling in ReactQuery

- polling refers to fetching data at regular intervals 
- eg: in stock market UI, you need to update UI in every sec 
- another config ```refetchInterval```
- default value is false 
- you can set it as a no in milliseconds that will result in refetch the query every x ms. 
- however, polling is paused if window loses focus 
but if you want to avoid this there is another config 
```refetchIntervalInBackground: true```

- using the above configs, you can provide an excelint UX where data changes every now and then 

## Fetching data based on user event 

1) Disable data fetching on component mount using config ```enabled: false```
2) Fetch data on click of button
<br/>
useQuery returns function refetch to fetch data manually, we do 

```
const { isLoading, data, isError, error, isFetching, refetch} =  useQuery('super-heroes', fetchSuperHeroes, {enabled: false, })

if (isLoading || isFetching) {
    return <h2>Loading...</h2>
}

if (isError) {
    return <h2>{error.message}</h2>
}
```

Inside component

```
<button onClick={refetch}>Refresh data </button>
```

## Callbacks with useQuery

- sometimes we want to perform a side-effect when query completes 
- eg: opening modal, navigating to different route, display toast notif 
- to do this, ReactQuery lets us specify success and error Callbacks as configs to useQuery hook 

1) we need two functions to be called when query succeeds or fails: onSuccess/onError

```
const onSuccess = data => {
    console.log({ data })
}

const onError = error => {
    console.log({ error })
} 

const { isLoading, data, isError, error, isFetching, refetch } = useQuery('super-heroes', fetchSuperHeroes,
{
    onSuccess: onSuccess,
    onError: onError,
})
```

## Data Transformation from useQuery hook 

- config ```select: ```
- it will receive data as an argument 

```
const { isLoading, data, isError, error, isFetching, refetch } = useQuery('super-heroes', fetchSuperHeroes,
    {
      onSuccess: onSuccess,
      onError: onError,
      select: (data) => {
        const names = data.data.map(hero => hero.name);
        return names;
      }
    })

return (
    <>
      {/* data below refers to the names array */}
      {data.map((name) => {
        return <div>{name}</div>
      })}
    </>
  )
```

- in the above eg, we change destructured data into an array 

## Custom Query Hook

Custom hook naming convention:
use <what it's retrieving>

```
import { useQuery } from "react-query"
import axios from "axios" 

const fetchSuperHeroes = () => {
    return axios.get('http://localhost:4000/superheroes')
}

export const useSuperHeroesData = (onSuccess, onError) => {
    return useQuery('super-heroes', fetchSuperHeroes,
        {
            onSuccess: onSuccess,
            onError: onError,
            select: (data) => {
                const names = data.data.map(hero => hero.name);
                return names;
            }
        })
}
```

## Posting data using useMutation

- for CUD operations we use Mutations 
- useMutation doesn't necessarily need a key like useQuery does
- parameters: Function that will post data to backend
```
const addSomething = (dataToBePosted) => {
    return axios.post('http://url', dataToBePosted)
} 
```
- useMutation returns 
1) mutate function that we have to call to make POST req 
2) flags like isLoading, isError

## Query invalidation 

- suppose you are adding something using a form and UI is out of date 
- we want React query to refetch data as soon as mutation succeeds 
- this feature is called Query invalidation 

### Implementing Query invalidation 

- Create instance of ```useQueryClient```, let's say queryClient
-  get hold of success callback of useMutation hook, the code in here will exceute as soon as mutation succeeds
- in the callback, 
we will first update the queryCache
```
const queryClient = useQueryClient();
return useMutation(addSomething, {
    onSuccess: (data)=> {

        // update the queryCache (append the mutation resp)
        queryClient.setQueryData('key-used-by-useQuery', (oldQueryData) => {
            return {
                ...oldQueryData,
                data: [...oldQueryData.data, data.data]
            }
        })

        queryClient.invalidateQueries('key-used-by-useQuery')
    }
})
```

## Axios Interceptor 

- let's say you have a file called ```axios-utils.js```

```
import axios from 'axios'

const client = axios.create({ baseURL: 'http://localhost:4000' })

export const request = ({ ...options }) => {

    // setting Auth bearer token
    client.defaults.headers.common.Authorization = `Bearer ${token}`

    const onSuccess = response => response
    const onError = error => {
        // optionaly catch errors and add additional logging here
        return error
    }

    return client(options).then(onSuccess).catch(onError)
}

```

- Now for every request the base URL will be ```http://localhost:4000``` and bearer token will be present in the header 

- Using the receptor 

```
import {request} from './utils/axios-utils'

const addSomething = (dataToBePosted) => {
    // without Interceptor
    return axios.post('http://url', dataToBePosted)

    // with Interceptor
    return request({url: '/endpt', method: 'post/get', data: whatever})
}
```