# Custom event thực sự trên DOM (CustomEvent)


## Detail
+ Đây là event native của browser, bạn tạo bằng new CustomEvent() và lắng nghe bằng addEventListener.
+ React có thể tương tác với nó thông qua useEffect.

```reacjs
// Component phát sự kiện
export function EventSender() {
  const sendEvent = () => {
    const event = new CustomEvent("myCustomEvent", {
      detail: { message: "Hello from EventSender" }
    });
    window.dispatchEvent(event);
  };

  return <button onClick={sendEvent}>Gửi Custom DOM Event</button>;
}

```

```reacjs
// Component lắng nghe sự kiện
import { useEffect } from "react";

export function EventListener() {
  useEffect(() => {
    const handler = (e) => {
      console.log("Nhận custom event:", e.detail);
    };
    window.addEventListener("myCustomEvent", handler);

    return () => {
      window.removeEventListener("myCustomEvent", handler);
    };
  }, []);

  return <div>Lắng nghe custom event...</div>;
}

```

## Case study
### Refresh token cross microsite
+ Để bài:
```
Triển khai refresh token cho nhiều site khác nhau trong reactjs.
bProWeb con host chứa widget.
1. Khi Widget  gọi API với token cũ → bị 401.
2. Widget bắn requestTokenRefresh qua window event.
3. bProWeb bắt được → gọi API refresh token → bắn tokenRefreshed với token mới.
4. Widget nhận token mới và gọi API lại
Widget sử dụng axios intercepter đển handler 401 và sẽ có timeout đợi refresh token 1s không có thì trả về lỗi và dựng lại. 
Dùng customEvent để triển khai giúp tôi giải pháp trên.
```
+ Nguyên tắc hoạt động
```
+ Widget khi API trả về 401 → dispatch sự kiện "requestTokenRefresh".
+ bProWeb (host) lắng nghe "requestTokenRefresh" → gọi API refresh token → dispatch "tokenRefreshed" kèm token mới.
+ Widget lắng nghe "tokenRefreshed" → cập nhật token → gọi lại API.
+ Nếu Widget đợi hơn 1 giây không nhận được token mới → trả lỗi và dựng lại.
```
+ Triển khai widget
```reactjs
// widgetAxios.js
import axios from "axios";

let accessToken = localStorage.getItem("accessToken");

function setAccessToken(token) {
  accessToken = token;
  localStorage.setItem("accessToken", token);
}

// Lắng nghe token mới từ bProWeb
window.addEventListener("tokenRefreshed", (event) => {
  const { token } = event.detail;
  if (token) {
    setAccessToken(token);
    console.log("Widget: Token mới nhận:", token);
  }
});

async function waitForToken(timeout = 1000) {
  return new Promise((resolve) => {
    let resolved = false;

    const handler = (event) => {
      resolved = true;
      window.removeEventListener("tokenRefreshed", handler);
      resolve(event.detail.token);
    };

    window.addEventListener("tokenRefreshed", handler);

    setTimeout(() => {
      if (!resolved) {
        window.removeEventListener("tokenRefreshed", handler);
        resolve(null);
      }
    }, timeout);
  });
}

const widgetAxios = axios.create({
  baseURL: "https://api.example.com",
});

widgetAxios.interceptors.request.use((config) => {
  if (accessToken) {
    config.headers.Authorization = `Bearer ${accessToken}`;
  }
  return config;
});

widgetAxios.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (error.response?.status === 401) {
      console.log("Widget: Token hết hạn, yêu cầu refresh...");

      let retryCount = 0;
      let newToken = null;

      while (retryCount < 3 && !newToken) {
        retryCount++;
        console.log(`Widget: Thử refresh token lần ${retryCount}...`);

        window.dispatchEvent(new CustomEvent("requestTokenRefresh"));
        newToken = await waitForToken(1000);

        if (newToken) {
          console.log("Widget: Nhận token mới, gọi lại API.");
          return widgetAxios(error.config);
        }
      }

      console.error("Widget: Không thể refresh token sau 3 lần thử.");
      return Promise.reject(error);
    }

    return Promise.reject(error);
  }
);

export default widgetAxios;

```
+ Triển khai bProWeb case detail
```reactjs
// bProWeb.js
import axios from "axios";

let refreshToken = localStorage.getItem("refreshToken");

async function refreshAccessToken() {
  console.log("bProWeb: Gọi API refresh token...");
  const response = await axios.post("https://api.example.com/auth/refresh", {
    refreshToken,
  });

  const { accessToken: newToken } = response.data;

  // Lưu vào localStorage của host
  localStorage.setItem("accessToken", newToken);

  // Gửi token mới cho widget
  window.dispatchEvent(
    new CustomEvent("tokenRefreshed", { detail: { token: newToken } })
  );

  console.log("bProWeb: Đã gửi token mới cho widget.");
}

window.addEventListener("requestTokenRefresh", () => {
  refreshAccessToken().catch((err) => {
    console.error("bProWeb: Lỗi refresh token", err);
    // Vẫn gửi sự kiện để widget biết là không thành công
    window.dispatchEvent(new CustomEvent("tokenRefreshed", { detail: { token: null } }));
  });
});


```

## Docs
1. https://developer.mozilla.org/en-US/docs/Web/API/CustomEvent/CustomEvent
