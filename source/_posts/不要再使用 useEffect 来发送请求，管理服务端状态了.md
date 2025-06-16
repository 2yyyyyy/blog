---
title: 不要再使用 useEffect 来发送请求，管理服务端状态了
date: 2025-06-12 10:31:33
tags: react
categories: 前端
---

在 react 中如何管理服务端状态？相信很多小伙伴的第一反应就是： 在 useEffect 里发送请求，然后结合 useState 来管理状态。

下面我们来看下，使用这种方式存在什么问题，以及如何解决这些问题，最后我们会看看其他更好的方式。

## 在useEffect里发送请求，然后结合useState来管理状态
通常我们会这样写：
```js App.tsx
function App() {
  const [users, setUsers] = useState([]);
  useEffect(() => {
    fetch("https://jsonplaceholder.typicode.com/users")
      .then((res) => res.json())
      .then((data) => setUsers(data));
  }, []);
  return (
    <ul>
      {users.map((user) => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}
```

这作为一个 demo 演示代码或许可行，但是以上代码存在如下问题：

1. 没有处理接口请求中的状态，用户不知道我们在获取数据
2. 没有处理接口请求中的错误，如果接口发生了异常，用户没有感知

我们修复上面的代码：

```js App.tsx
function App() {
  const [users, setUsers] = useState();
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState(null);
  useEffect(() => {
    setIsLoading(true);
    fetch("https://jsonplaceholder.typicode.com/users")
      .then((res) => {
        if (!res.ok) {
          throw new Error("Failed to fetch users");
        }
        return res.json();
      })
      .then((data) => {
        setUsers(data);
        setIsLoading(false);
        setError(null);
      })
      .catch((error) => {
        setError(error);
        setIsLoading(false);
      });
  }, []);
  if (isLoading) {
    return <div>Loading...</div>;
  }
  if (error) {
    return <div>Error: {error.message}</div>;
  }
  return (
    <ul>
      {users.map((user) => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}
```

乍看一下，虽然多个 **useState** 的引入，让代码变得更加复杂，但是你可能还是能接受的。 但你的老板找到你，想让你加上搜索功能，于是乎你可能会写这样的代码：

```js App.tsx
function App() {
  const [users, setUsers] = useState([]);
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState(null);
  const [search, setSearch] = useState("");
  useEffect(() => {
    setIsLoading(true);
    fetch(`https://jsonplaceholder.typicode.com/users?q=${search}`)
      .then((res) => {
        if (!res.ok) {
          throw new Error("Failed to fetch users");
        }
        return res.json();
      })
      .then((data) => {
        setUsers(data);
        setIsLoading(false);
        setError(null);
      })
      .catch((error) => {
        setError(error);
        setIsLoading(false);
      });
  }, [search]);
  if (error) {
    return <div>Error: {error.message}</div>;
  }
  return (
    <div>
      <input
        type="text"
        value={search}
        onChange={(e)=> setSearch(e.target.value)}
      />
      {isLoading ? (
        <div>Loading...</div>
      ) : (
        <ul>
          {users.map((user) => (
            <li key={user.id}>{user.name}</li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

这段代码虽然看上去还可以，但是其实存在一个问题，当我们每次输入的时候，都会发送一个请求，假设我们先输入 a，然后输入 b，那么就会以 a 和 ab 为查询字符串分别发送两次请求，而服务端接口的速度快慢是不一定的，如果请求 a 响应比 请求 ab 更慢，那么我们的页面上就会先显示 ab 请求对应的结果， 然后再显示 a 请求对应的结果。这里存在如下问题:

1. 最终结果与 ui 不符，我们希望看到的是 ab 请求对应的结果。

如何解决这个问题呢？

你可能会想到，我们能不能对请求做个 debounce，即在输入时延迟一会儿再发送请求，这样就不会发送两次请求了。但 debounce 解决不了根本的问题， 因为服务端的接口快慢是不一定的，debounce 之后，用户输入稍微有些停顿，可能还是会发送两次请求，还是会造成多个请求竞争的问题。

而事实上我们要做的是，只要有新的请求发出，那么就取消之前的请求或者忽略之前的请求。这样就不会出现多个请求竞争的问题了。我们可以从 [react 的官网](https://react.dev/learn/synchronizing-with-effects#fetching-data)上找到类似的解决方案。

```js App.tsx
function App() {
  const [users, setUsers] = useState();
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState(null);
  const [search, setSearch] = useState("");
  useEffect(() => {
    let ignore = false;
    setIsLoading(true);
    fetch(`https://jsonplaceholder.typicode.com/users?q=${search}`)
      .then((res) => {
        if (!res.ok) {
          throw new Error("Failed to fetch users");
        }
        return res.json();
      })
      .then((data) => {
        if (ignore) return;
        setUsers(data);
        setIsLoading(false);
        setError(null);
      })
      .catch((error) => {
        if (ignore) return;
        setError(error);
        setIsLoading(false);
      });
    return () => {
      ignore = true;
    };
  }, [search]);
  if (error) {
    return <div>Error: {error.message}</div>;
  }
  return (
    <div>
      <input
        type="text"
        value={search}
        onChange={(e)=> setSearch(e.target.value)}
      />
      {isLoading ? (
        <div>Loading...</div>
      ) : (
        <ul>
          {users.map((user) => (
            <li key={user.id}>{user.name}</li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

由于 react 会在每次 执行 effect 之前去执行 cleanup，所以我们可以在 fetch 的 then 里做个 ignore 判断，如果 ignore 是 true，那么就不会执行 setState。

此时你会发现我们处理请求的代码已经越来越复杂了，而且这段代码里解决的问题，可能是我们每个请求都需要处理的，于是你可能会想到抽出一个公用的自定义 hook。

```js useQuery.ts
export function useQuery({ url }) {
  const [data, setData] = useState();
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState(null);
  useEffect(() => {
    let ignore = false;
    setIsLoading(true);
    fetch(url)
      .then((res) => {
        if (!res.ok) {
          throw new Error("Failed to fetch users");
        }
        return res.json();
      })
      .then((data) => {
        if (ignore) return;
        setData(data);
        setIsLoading(false);
        setError(null);
      })
      .catch((error) => {
        if (ignore) return;
        setError(error);
        setIsLoading(false);
      });
    return () => {
      ignore = true;
    };
  }, [url]);
  return {
    data,
    isLoading,
    error,
  };
}
```

```js App.tsx
import { useQuery } from "./useQuery";
function App() {
  const [search, setSearch] = useState("");
  const {
    data: users,
    isLoading,
    error,
  } = useQuery({
    url: `https://jsonplaceholder.typicode.com/users?q=${search}`,
  });
  if (isLoading) {
    return <div>Loading...</div>;
  }
  if (error) {
    return <div>Error: {error.message}</div>;
  }
  return (
    <div>
      <input
        type="text"
        value={search}
        onChange={(e)=> setSearch(e.target.value)}
      />
      {isLoading ? (
        <div>Loading...</div>
      ) : (
        <ul>
          {users.map((user) => (
            <li key={user.id}>{user.name}</li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

好了，我们的自定义 hook 完成了，看上去很不错，但是如果你在项目中大量使用它，你就会发现它可能会遇到如下问题:

1. 请求去重。我们的 useQuery 没有做请求去重，如果我们在多个组件中使用 useQuery, 传递同一个 url 参数，会发送多次请求，这其实是没有必要的，对服务端资源来说是一种浪费。
2. 请求缓存。我们的 useQuery 没有做缓存。对于一个追求高性能体验的客户端应用程序而言，缓存是必不可少的，因为我们并不是所有场景都要求非常实时的数据。而使用缓存可以帮助我们大大提高用户体验。但是我们知道在程序的世界里有句说法: 缓存和命名 是最难的。
3. 重新请求的机制。我们是否需要在用户离开页面一端时间再次返回页面之后（也许是去上了个厕所， 喝个咖啡， 锁屏回来重新打开屏幕），重新请求刷新数据？
4. 我们的 useQuery 内部将 fetch 封装进来了。这个耦合太大了，其实只需要传递一个 queryFn 和 queryKey 就可以了，queryFn 里去返回一个 promise，这里就又会牵扯到 queryKey 结构的问题，因为我们要监听 queryKey 的变化。我们如何高效比对两个 queryKey 是否相同？

## 使用三方库

好了，说了那么多，无非是想说明，维护服务端请求状态确实是一件很复杂的事情，如果你追求极致的用户体验，那么我的建议是使用三方库来完成这个工作，目前在这个领域使用最广泛的是 [react-query](https://tanstack.com/query/latest), 他提供了如下特性:

1. 声明式的 api，我们只需声明一个 query，改变 query 的参数，就会触发 query 的重新执行。
2. 自动处理缓存
3. 自动处理接口调用去重
4. 对后端不感知。
5. 自动重试
6. 预加载
7. 请求取消机制
8. 其他

如果你想要了解更多的细节，可以参考 [react-query 的文档](https://tanstack.com/query/latest/docs/framework/react/overview)。

我们使用 react-query 对我们的示例程序改写:

```js 
import {
  QueryClient,
  QueryClientProvider,
  useQuery,
} from "@tanstack/react-query";

const queryClient = new QueryClient();

export default function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <Example />
    </QueryClientProvider>
  );
}

function Example() {
  const [search, setSearch] = useState("");
  const { data, isPending, error } = useQuery({
    queryKey: ["users", search],
    queryFn: () =>
      fetch(`https://jsonplaceholder.typicode.com/users?q=${search}`).then(
        (res) => {
          if (!res.ok) {
            throw new Error("Failed to fetch users");
          }
          return res.json();
        }
      ),
  });
  if (error) {
    return <div>Error: {error.message}</div>;
  }
  return (
    <div>
      <input
        type="text"
        value={search}
        onChange={(e)=> setSearch(e.target.value)}
      />
      {isPending ? (
        <div>Loading...</div>
      ) : (
        <ul>
          {users.map((user) => (
            <li key={user.id}>{user.name}</li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

好了，改造完成，看起来如此优雅的 api，希望在以后的项目中能够有机会去使用它。
