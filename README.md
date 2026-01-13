# flashcard
flashcard
import { useState } from "react";
import {
  Upload,
  FileText,
  AlertCircle,
  CheckCircle,
  Download,
} from "lucide-react";

interface CSVUploadProps {
  onUpload: (
    cards: Array<{
      front: string;
      back: string;
      description?: string;
    }>,
  ) => void;
}

export function CSVUpload({ onUpload }: CSVUploadProps) {
  const [error, setError] = useState<string>("");
  const [success, setSuccess] = useState<string>("");
  const [isDragging, setIsDragging] = useState(false);

  const parseCSV = (
    text: string,
  ): Array<{
    front: string;
    back: string;
    description?: string;
  }> => {
    const lines = text
      .split("\n")
      .filter((line) => line.trim());

    if (lines.length < 2) {
      throw new Error(
        "CSV 파일이 비어있거나 데이터가 없습니다.",
      );
    }

    // 첫 번째 줄은 헤더로 간주 (선택사항)
    const firstLine = lines[0].toLowerCase();
    const hasHeader =
      firstLine.includes("front") ||
      firstLine.includes("앞면") ||
      firstLine.includes("질문");
    const dataLines = hasHeader ? lines.slice(1) : lines;

    const cards: Array<{
      front: string;
      back: string;
      description?: string;
    }> = [];

    dataLines.forEach((line, index) => {
      // CSV 파싱: 쉼표로 구분하되, 따옴표 안의 쉼표는 무시
      const regex = /(?:,|^)(?:"([^"]*)"|([^",]*))/g;
      const values: string[] = [];
      let match;

      while ((match = regex.exec(line)) !== null) {
        values.push(
          match[1] !== undefined ? match[1] : match[2],
        );
      }

      // 빈 값 제거
      const cleanValues = values.filter(
        (v) => v !== undefined && v !== null,
      );

      if (cleanValues.length < 2) {
        throw new Error(
          `${hasHeader ? index + 2 : index + 1}번째 줄: 최소 2개의 값(앞면, 뒷면)이 필요합니다.`,
        );
      }

      const [front, back, description] = cleanValues;

      if (!front.trim() || !back.trim()) {
        throw new Error(
          `${hasHeader ? index + 2 : index + 1}번째 줄: 앞면과 뒷면은 비어있을 수 없습니다.`,
        );
      }

      cards.push({
        front: front.trim(),
        back: back.trim(),
        description: description?.trim() || undefined,
      });
    });

    return cards;
  };

  const handleFile = (file: File) => {
    setError("");
    setSuccess("");

    if (!file.name.endsWith(".csv")) {
      setError("CSV 파일만 업로드 가능합니다.");
      return;
    }

    const reader = new FileReader();
    reader.onload = (e) => {
      try {
        const text = e.target?.result as string;
        const cards = parseCSV(text);

        if (cards.length === 0) {
          setError("업로드할 카드가 없습니다.");
          return;
        }

        onUpload(cards);
        setSuccess(
          `${cards.length}개의 카드가 성공적으로 추가되었습니다!`,
        );
      } catch (err) {
        setError(
          err instanceof Error
            ? err.message
            : "파일 처리 중 오류가 발생했습니다.",
        );
      }
    };
    reader.readAsText(file);
  };

  const handleFileInput = (
    e: React.ChangeEvent<HTMLInputElement>,
  ) => {
    const file = e.target.files?.[0];
    if (file) {
      handleFile(file);
    }
    // Reset input
    e.target.value = "";
  };

  const handleDrop = (e: React.DragEvent) => {
    e.preventDefault();
    setIsDragging(false);

    const file = e.dataTransfer.files?.[0];
    if (file) {
      handleFile(file);
    }
  };

  const handleDragOver = (e: React.DragEvent) => {
    e.preventDefault();
    setIsDragging(true);
  };

  const handleDragLeave = () => {
    setIsDragging(false);
  };

  const downloadTemplate = () => {
    const template =
      'front,back,description\napple,사과,"빨간색 과일, 영양가가 높음"\nbanana,바나나,노란색 과일\norange,오렌지,';
    const blob = new Blob([template], {
      type: "text/csv;charset=utf-8;",
    });
    const link = document.createElement("a");
    link.href = URL.createObjectURL(blob);
    link.download = "flashcard_template.csv";
    link.click();
  };

  return (
    <div className="space-y-4">
      {/* Instructions */}
      <div className="bg-lime-50 border border-lime-200 rounded-lg p-3 sm:p-4">
        <div className="flex items-start gap-2 sm:gap-3">
          <FileText className="w-4 h-4 sm:w-5 sm:h-5 text-lime-700 mt-0.5 flex-shrink-0" />
          <div className="flex-1">
            <h3 className="text-sm sm:text-base font-semibold text-lime-900 mb-2">
              CSV 파일 형식 안내
            </h3>
            <ul className="text-xs sm:text-sm text-lime-800 space-y-1">
              <li>• 첫 번째 열: 앞면 (질문 또는 단어)</li>
              <li>• 두 번째 열: 뒷면 (답 또는 뜻)</li>
              <li>• 세 번째 열: 설명 (선택사항)</li>
              <li>
                • 첫 번째 줄은 헤더로 사용 가능 (선택사항)
              </li>
              <li>
                • 쉼표가 포함된 내용은 큰따옴표로 감싸주세요
              </li>
            </ul>
            <button
              onClick={downloadTemplate}
              className="mt-3 flex items-center gap-1.5 text-xs sm:text-sm text-lime-700 hover:text-lime-900 font-medium"
            >
              <Download className="w-3 h-3 sm:w-4 sm:h-4" />
              샘플 템플릿 다운로드
            </button>
          </div>
        </div>
      </div>

      {/* Upload Area */}
      <div
        onDrop={handleDrop}
        onDragOver={handleDragOver}
        onDragLeave={handleDragLeave}
        className={`relative border-2 border-dashed rounded-lg p-6 sm:p-8 text-center transition-colors ${
          isDragging
            ? "border-emerald-500 bg-emerald-50"
            : "border-gray-300 bg-gray-50 hover:border-emerald-400"
        }`}
      >
        <input
          type="file"
          accept=".csv"
          onChange={handleFileInput}
          className="absolute inset-0 w-full h-full opacity-0 cursor-pointer"
        />
        <div className="flex flex-col items-center gap-3">
          <Upload
            className={`w-10 h-10 sm:w-12 sm:h-12 ${isDragging ? "text-emerald-600" : "text-gray-400"}`}
          />
          <div>
            <p className="text-sm sm:text-lg font-medium text-gray-700">
              CSV 파일을 여기에 드래그하거나 클릭하여 선택
            </p>
            <p className="text-xs sm:text-sm text-gray-500 mt-1">
              .csv 파일만 지원됩니다
            </p>
          </div>
        </div>
      </div>

      {/* Error Message */}
      {error && (
        <div className="flex items-start gap-2 sm:gap-3 bg-red-50 border border-red-200 rounded-lg p-3 sm:p-4">
          <AlertCircle className="w-4 h-4 sm:w-5 sm:h-5 text-red-600 mt-0.5 flex-shrink-0" />
          <div className="flex-1">
            <p className="text-xs sm:text-sm font-medium text-red-900">
              오류 발생
            </p>
            <p className="text-xs sm:text-sm text-red-700 mt-1 break-words">
              {error}
            </p>
          </div>
        </div>
      )}

      {/* Success Message */}
      {success && (
        <div className="flex items-start gap-2 sm:gap-3 bg-green-50 border border-green-200 rounded-lg p-3 sm:p-4">
          <CheckCircle className="w-4 h-4 sm:w-5 sm:h-5 text-green-600 mt-0.5 flex-shrink-0" />
          <div className="flex-1">
            <p className="text-xs sm:text-sm font-medium text-green-900">
              업로드 성공
            </p>
            <p className="text-xs sm:text-sm text-green-700 mt-1">
              {success}
            </p>
          </div>
        </div>
      )}
    </div>
  );
}
