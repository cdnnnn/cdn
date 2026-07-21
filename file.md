export interface CaptionLine {
    id: number;
    start_time: string;
    end_time: string;
    duration: string;
    text: string;
}

interface CaptionsResponse {
    total_segments: number;
    captions: CaptionLine[];
    is_vtt: boolean;
}

export async function fetchCaptions(fileId: number): Promise<{ lines: CaptionLine[]; is_vtt: boolean }> {
    const { data } = await api.post<CaptionsResponse>('/stt/get_captions', {
        fileID: fileId,
    });
    return {
        lines: data.captions ?? [],
        is_vtt: !!data.is_vtt,
    };
}
