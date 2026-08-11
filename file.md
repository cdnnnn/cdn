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
      // Category is set to the eval type per API contract.
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
