import axios from 'axios';
import type { AxiosError, InternalAxiosRequestConfig } from 'axios';
import { store } from '../store/store';
import { logout } from '../store/slices/authSlice';

const TOKEN_STORAGE_KEY = 'semcoeval_token';

export const tokenStorage = {
  get: (): string | null => localStorage.getItem(TOKEN_STORAGE_KEY),
  set: (token: string): void => localStorage.setItem(TOKEN_STORAGE_KEY, token),
  clear: (): void => localStorage.removeItem(TOKEN_STORAGE_KEY),
};

export const api = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL || '/api',
  timeout: 20000,
  headers: {
    'Content-Type': 'application/json',
  },
});

// ---- Request interceptor: attach bearer token ----
api.interceptors.request.use(
  (config: InternalAxiosRequestConfig) => {
    const token = tokenStorage.get();
    if (token) {
      config.headers = config.headers ?? {};
      config.headers.Authorization = `Bearer ${token}`;
    }

    // For FormData bodies (file uploads), drop the default JSON
    // Content-Type so the browser can set the correct multipart
    // boundary itself. Setting this header to undefined/omitting it
    // per-call isn't reliable across axios versions once it's already
    // merged in from the instance defaults, so delete it here instead.
    if (typeof FormData !== 'undefined' && config.data instanceof FormData && config.headers) {
      delete config.headers['Content-Type'];
    }

    return config;
  },
  (error: AxiosError) => Promise.reject(error)
);

// ---- Response interceptor: normalize errors + handle 401 ----
let isLoggingOut = false;

api.interceptors.response.use(
  (response) => response,
  (error: AxiosError<{ message?: string }>) => {
    const status = error.response?.status;

    if (status === 401 && !isLoggingOut) {
      isLoggingOut = true;
      tokenStorage.clear();
      store.dispatch(logout());
      isLoggingOut = false;
    }

    const message =
      error.response?.data?.message ||
      error.message ||
      'Something went wrong while talking to the server.';

    return Promise.reject({ ...error, message });
  }
);

export default api;




















import { apiClient } from '../client';
import type { Dataset } from '../../store/slices/datasetsSlice';

interface DatasetsResponse {
  datasets: Dataset[];
  total: number;
}

export interface UploadDatasetParams {
  file: File;
  name: string;
  description: string;
  evalType: string;
}

// Extensions routed to POST /datasets/upload
const STRUCTURED_EXTENSIONS = ['json', 'arrow', 'parquet'];
// Extension routed to POST /datasets/upload-jsonl
const JSONL_EXTENSION = 'jsonl';

export const SUPPORTED_UPLOAD_EXTENSIONS = [...STRUCTURED_EXTENSIONS, JSONL_EXTENSION];

function getExtension(filename: string): string {
  const idx = filename.lastIndexOf('.');
  return idx >= 0 ? filename.slice(idx + 1).toLowerCase() : '';
}

export const datasetsApi = {
  // GET /datasets?eval_type={type}
  list: async (evalType: string): Promise<Dataset[]> => {
    const { data } = await apiClient.get<DatasetsResponse>('/datasets', { params: { eval_type: evalType } });
    return data.datasets;
  },

  // Routes to POST /datasets/upload (.json, .arrow, .parquet) or
  // POST /datasets/upload-jsonl (.jsonl) based on the file extension.
  upload: async ({ file, name, description, evalType }: UploadDatasetParams): Promise<Dataset> => {
    const ext = getExtension(file.name);

    if (!SUPPORTED_UPLOAD_EXTENSIONS.includes(ext)) {
      throw new Error('Unsupported file type. Please upload a .json, .jsonl, .arrow, or .parquet file.');
    }

    const formData = new FormData();
    formData.append('file', file);

    if (ext === JSONL_EXTENSION) {
      // Category is set to the eval type per API contract. The axios
      // request interceptor drops the default JSON Content-Type for
      // FormData bodies, so the browser can attach the correct
      // multipart boundary itself.
      const { data } = await apiClient.post<Dataset>('/datasets/upload-jsonl', formData, {
        params: { name, eval_type: evalType, category: evalType, description },
      });
      return data;
    }

    const { data } = await apiClient.post<Dataset>('/datasets/upload', formData, {
      params: { eval_type: evalType, name, description },
    });
    return data;
  },
};


