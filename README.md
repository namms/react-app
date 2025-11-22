

   ```json
   {
     "username": "john",
     "permissions": ["view_dashboard", "view_reports"],
     "group": "admin"
   }
   ```
3. Frontend stores this **decoded info only** in Redux.

This is the safest and most correct method.

---

# ✅ 1. **Redux Toolkit Auth Slice**

**src/store/authSlice.ts**

```ts
import { createSlice, createAsyncThunk } from "@reduxjs/toolkit";
import axios from "../utils/axios";

export interface AuthState {
  username: string | null;
  permissions: string[];
  group: string | null;
  loading: boolean;
  isAuthenticated: boolean;
}

const initialState: AuthState = {
  username: null,
  permissions: [],
  group: null,
  loading: true,
  isAuthenticated: false
};

// Fetch user info by reading HttpOnly cookie (JWT) on backend
export const fetchUser = createAsyncThunk(
  "auth/fetchUser",
  async (_, { rejectWithValue }) => {
    try {
      const res = await axios.get("/auth/me", { withCredentials: true });
      return res.data;
    } catch (err: any) {
      return rejectWithValue(err.response?.data || "Unauthorized");
    }
  }
);

const authSlice = createSlice({
  name: "auth",
  initialState,
  reducers: {
    clearAuth: (state) => {
      state.username = null;
      state.permissions = [];
      state.group = null;
      state.isAuthenticated = false;
    }
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchUser.pending, (state) => {
        state.loading = true;
      })
      .addCase(fetchUser.fulfilled, (state, action) => {
        state.username = action.payload.username;
        state.permissions = action.payload.permissions;
        state.group = action.payload.group;
        state.isAuthenticated = true;
        state.loading = false;
      })
      .addCase(fetchUser.rejected, (state) => {
        state.loading = false;
        state.isAuthenticated = false;
      });
  }
});

export const { clearAuth } = authSlice.actions;
export default authSlice.reducer;
```

---

# ✅ 2. **Global Store Setup**

**src/store/store.ts**

```ts
import { configureStore } from "@reduxjs/toolkit";
import authReducer from "./authSlice";

export const store = configureStore({
  reducer: {
    auth: authReducer
  }
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

---

# ✅ 3. **Axios Instance**

Cookies are automatically included.

**src/utils/axios.ts**

```ts
import axios from "axios";

const instance = axios.create({
  baseURL: "https://api.example.com",
  withCredentials: true // enables HttpOnly Cookie exchange
});

export default instance;
```

---

# ✅ 4. **App Initialization (load auth on navigation)**

We load the user **on first render**, BEFORE pages load.

**src/App.tsx**

```tsx
import { useEffect } from "react";
import { useDispatch, useSelector } from "react-redux";
import { fetchUser } from "./store/authSlice";
import { RootState } from "./store/store";
import AppRoutes from "./routes/AppRoutes";

export default function App() {
  const dispatch = useDispatch();
  const loading = useSelector((s: RootState) => s.auth.loading);

  useEffect(() => {
    dispatch(fetchUser() as any);
  }, [dispatch]);

  if (loading) return <div>Loading authentication...</div>;

  return <AppRoutes />;
}
```

---

# ✅ 5. **Protected Route**

**src/routes/ProtectedRoute.tsx**

```tsx
import React from 'react';
import { Navigate, useLocation } from 'react-router-dom';
import { useSelector } from 'react-redux';
import { RootState } from '../store';
import { UserRole } from '../types';

interface ProtectedRouteProps {
  children: React.ReactNode;
  allowedRoles: UserRole[];
}

const ProtectedRoute: React.FC<ProtectedRouteProps> = ({ children, allowedRoles }) => {
  const { isAuthenticated, user } = useSelector((state: RootState) => state.auth);
  const location = useLocation();

  if (!isAuthenticated || !user) {
    // Redirect to login, saving the location they were trying to access
    return <Navigate to="/login" state={{ from: location }} replace />;
  }

  if (!allowedRoles.includes(user.role)) {
    // Redirect to unauthorized page if role doesn't match
    return <Navigate to="/unauthorized" replace />;
  }

  return <>{children}</>;
};

export default ProtectedRoute;
```

---

# ✅ 6. **Permission Route**

**src/routes/PermissionRoute.tsx**

```tsx
import { useSelector } from "react-redux";
import { Navigate } from "react-router-dom";
import { RootState } from "../store/store";

export default function PermissionRoute({ children, permission }: any) {
  const { permissions } = useSelector((s: RootState) => s.auth);

  if (!permissions.includes(permission)) {
    return <Navigate to="/unauthorized" replace />;
  }

  return children;
}
```

---

# ✅ 7. **Route Config w/ Permission-Based Screens**

**src/routes/AppRoutes.tsx**

```tsx
import { BrowserRouter, Routes, Route } from "react-router-dom";
import ProtectedRoute from "./ProtectedRoute";
import PermissionRoute from "./PermissionRoute";

import Dashboard from "../pages/Dashboard";
import Reports from "../pages/Reports";
import Login from "../pages/Login";

export default function AppRoutes() {
  return (
    <BrowserRouter>
      <Routes>

        <Route path="/login" element={<Login />} />

        <Route
          path="/dashboard"
          element={
            <ProtectedRoute>
              <PermissionRoute permission="view_dashboard">
                <Dashboard />
              </PermissionRoute>
            </ProtectedRoute>
          }
        />

        <Route
          path="/reports"
          element={
            <ProtectedRoute>
              <PermissionRoute permission="view_reports">
                <Reports />
              </PermissionRoute>
            </ProtectedRoute>
          }
        />

        <Route path="*" element={<ProtectedRoute><div>404</div></ProtectedRoute>} />
      </Routes>
    </BrowserRouter>
  );
}
```

---

# ✅ 8. **Mega Menu (show only allowed)**

**src/components/MegaMenu.tsx**

```tsx
import { useSelector } from "react-redux";
import { RootState } from "../store/store";

export default function MegaMenu() {
  const { permissions } = useSelector((s: RootState) => s.auth);

  const items = [
    { label: "Dashboard", permission: "view_dashboard", path: "/dashboard" },
    { label: "Reports", permission: "view_reports", path: "/reports" },
    { label: "Admin", permission: "admin_access", path: "/admin" }
  ];

  const allowed = items.filter(i => permissions.includes(i.permission));

  return (
    <ul>
      {allowed.map(item => (
        <li key={item.path}>
          <a href={item.path}>{item.label}</a>
        </li>
      ))}
    </ul>
  );
}
```
