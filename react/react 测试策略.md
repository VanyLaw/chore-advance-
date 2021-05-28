# 创建/清理 #
对于每个测试，我们通常希望将 React 树渲染给附加到 document的 DOM 元素。这点很重要，以便它可以接收 DOM 事件。当测试结束时，我们需要“清理”并从 document 中卸载树。
常见的方法是使用一对 beforeEach 和 afterEach 块。

```
import { unmountComponentAtNode } from "react-dom";

let container = null;
beforeEach(() => {
  // 创建一个 DOM 元素作为渲染目标
  container = document.createElement("div");
  document.body.appendChild(container);
});

afterEach(() => {
  // 退出时进行清理
  unmountComponentAtNode(container);
  container.remove();
  container = null;
});
```

# act() #
在编写 UI 测试时，可以将渲染、用户事件或数据获取等任务视为与用户界面交互的“单元”。react-dom/test-utils 提供了一个名为 act() 的 helper，它确保在进行任何断言之前，与这些“单元”相关的所有更新都已处理并应用于 DOM：
```
act(() => {
  // 渲染组件
});
// 进行断言
```

# 渲染测试 #
```
// hello.js

import React from "react";

export default function Hello(props) {
  if (props.name) {
    return <h1>你好，{props.name}！</h1>;
  } else {
    return <span>嘿，陌生人</span>;
  }
}
```
我们可以为这个组件编写测试：

```
// hello.test.js

import React from "react";
import { render, unmountComponentAtNode } from "react-dom";
import { act } from "react-dom/test-utils";

import Hello from "./hello";

let container = null;
beforeEach(() => {
  // 创建一个 DOM 元素作为渲染目标
  container = document.createElement("div");
  document.body.appendChild(container);
});

afterEach(() => {
  // 退出时进行清理
  unmountComponentAtNode(container);
  container.remove();
  container = null;
});

it("渲染有或无名称", () => {
  act(() => {
    render(<Hello />, container);
  });
  expect(container.textContent).toBe("嘿，陌生人");

  act(() => {
    render(<Hello name="Jenny" />, container);
  });
  expect(container.textContent).toBe("你好，Jenny！");

  act(() => {
    render(<Hello name="Margaret" />, container);
  });
  expect(container.textContent).toBe("你好，Margaret！");
});
```

# 数据获取 #
你可以使用假数据来 mock 请求，而不是在所有测试中调用真正的 API。使用“假”数据 mock 数据获取可以防止由于后端不可用而导致的测试不稳定，并使它们运行得更快。注意：你可能仍然希望使用一个“端到端”的框架来运行测试子集，该框架可显示整个应用程序是否一起工作。
```
// user.js

import React, { useState, useEffect } from "react";

export default function User(props) {
  const [user, setUser] = useState(null);

  async function fetchUserData(id) {
    const response = await fetch("/" + id);
    setUser(await response.json());
  }

  useEffect(() => {
    fetchUserData(props.id);
  }, [props.id]);

  if (!user) {
    return "加载中...";
  }

  return (
    <details>
      <summary>{user.name}</summary>
      <strong>{user.age}</strong> 岁
      <br />
      住在 {user.address}
    </details>
  );
}
```
我们可以为它编写测试：
```
// user.test.js

import React from "react";
import { render, unmountComponentAtNode } from "react-dom";
import { act } from "react-dom/test-utils";
import User from "./user";

let container = null;
beforeEach(() => {
  // 创建一个 DOM 元素作为渲染目标
  container = document.createElement("div");
  document.body.appendChild(container);
});

afterEach(() => {
  // 退出时进行清理
  unmountComponentAtNode(container);
  container.remove();
  container = null;
});

it("渲染用户数据", async () => {
  const fakeUser = {
    name: "Joni Baez",
    age: "32",
    address: "123, Charming Avenue"
  };
  jest.spyOn(global, "fetch").mockImplementation(() =>
    Promise.resolve({
      json: () => Promise.resolve(fakeUser)
    })
  );

  // 使用异步的 act 应用执行成功的 promise
  await act(async () => {
    render(<User id="123" />, container);
  });

  expect(container.querySelector("summary").textContent).toBe(fakeUser.name);
  expect(container.querySelector("strong").textContent).toBe(fakeUser.age);
  expect(container.textContent).toContain(fakeUser.address);

  // 清理 mock 以确保测试完全隔离
  global.fetch.mockRestore();
});
```

https://zh-hans.reactjs.org/docs/testing-recipes.html



