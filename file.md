//uiSlice.ts
import { createSlice, type PayloadAction } from '@reduxjs/toolkit';

export interface UiState {
  sidebarCollapsed: boolean;
  activeNavItem: string;
}

const initialState: UiState = {
  sidebarCollapsed: false,
  activeNavItem: 'dashboard',
};

const uiSlice = createSlice({
  name: 'ui',
  initialState,
  reducers: {
    toggleSidebar: (state) => {
      state.sidebarCollapsed = !state.sidebarCollapsed;
    },
    setSidebarCollapsed: (state, action: PayloadAction<boolean>) => {
      state.sidebarCollapsed = action.payload;
    },
    setActiveNavItem: (state, action: PayloadAction<string>) => {
      state.activeNavItem = action.payload;
    },
  },
});

export const { toggleSidebar, setSidebarCollapsed, setActiveNavItem } = uiSlice.actions;
export default uiSlice.reducer;


















//userSlice.ts
import { createSlice } from '@reduxjs/toolkit';

export interface User {
  name: string;
  role: string;
}

interface UserState {
  user: User;
}

const initialState: UserState = {
  user: { name: 'Alex Carter', role: 'Workspace Admin' },
};

const userSlice = createSlice({
  name: 'user',
  initialState,
  reducers: {},
});

export default userSlice.reducer;












//hooks.ts
import { useDispatch, useSelector, type TypedUseSelectorHook } from 'react-redux';
import type { RootState, AppDispatch } from './index';

export const useAppDispatch = () => useDispatch<AppDispatch>();
export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector;








//index.ts
import { configureStore } from '@reduxjs/toolkit';
import uiReducer from './slices/uiSlice';
import userReducer from './slices/userSlice';

export const store = configureStore({
  reducer: {
    ui: uiReducer,
    user: userReducer,
  },
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;

export default store;
