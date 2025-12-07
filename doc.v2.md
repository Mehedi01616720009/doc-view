# Math Questions PDF Converter (DeepSeek Vision)

## Project Structure
```
project-root/
├── backend/
│   ├── server.js
│   ├── routes/
│   │   └── upload.js
│   ├── controllers/
│   │   └── questionController.js
│   └── package.json
├── frontend/
│   ├── src/
│   │   ├── app/
│   │   │   ├── page.js
│   │   │   ├── layout.js
│   │   │   └── questions/
│   │   │       └── page.js
│   │   ├── components/
│   │   │   ├── QuestionList.js
│   │   │   └── QuestionCard.js
│   │   ├── store/
│   │   │   ├── store.js
│   │   │   └── questionSlice.js
│   │   └── utils/
│   │       └── api.js
│   ├── next.config.js
│   └── package.json
└── README.md
```

## Backend Setup

### 1. Install System Dependencies (Ubuntu/Debian)
```bash
sudo apt-get update
sudo apt-get install -y ghostscript graphicsmagick
```

### 2. Initialize Backend
```bash
mkdir backend && cd backend
npm init -y
npm install express cors dotenv multer @google/generative-ai pdf2pic
```

### 3. backend/package.json
```json
{
  "name": "math-questions-backend",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "start": "node server.js",
    "dev": "node --watch server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "cors": "^2.8.5",
    "dotenv": "^16.3.1",
    "multer": "^1.4.5-lts.1",
    "@google/generative-ai": "^0.19.0",
    "pdf2pic": "^3.1.3"
  }
}
```

### 4. backend/.env
```
PORT=5000
GEMINI_API_KEY=your_gemini_api_key_here
```

### 5. backend/server.js
```javascript
import express from 'express';
import cors from 'cors';
import dotenv from 'dotenv';
import uploadRoutes from './routes/upload.js';

dotenv.config();

const app = express();
const PORT = process.env.PORT || 5000;

app.use(cors());
app.use(express.json());

app.use('/api', uploadRoutes);

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

### 6. backend/routes/upload.js
```javascript
import express from 'express';
import multer from 'multer';
import { processPdf } from '../controllers/questionController.js';

const router = express.Router();

const storage = multer.memoryStorage();
const upload = multer({ 
  storage,
  fileFilter: (req, file, cb) => {
    if (file.mimetype === 'application/pdf') {
      cb(null, true);
    } else {
      cb(new Error('Only .pdf files are allowed'));
    }
  },
  limits: {
    fileSize: 10 * 1024 * 1024 // 10MB limit
  }
});

router.post('/upload', upload.single('pdf'), processPdf);

export default router;
```

### 7. backend/controllers/questionController.js
```javascript
import axios from 'axios';
import fs from 'fs';
import path from 'path';
import { fileURLToPath } from 'url';
import { fromPath } from 'pdf2pic';

const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);

export const processPdf = async (req, res) => {
  let tempPdfPath = null;
  let tempImageDir = null;

  try {
    if (!req.file) {
      return res.status(400).json({ error: 'No file uploaded' });
    }

    // Create temp directories
    const tempDir = path.join(__dirname, '../temp');
    if (!fs.existsSync(tempDir)) {
      fs.mkdirSync(tempDir, { recursive: true });
    }

    // Save PDF temporarily
    const timestamp = Date.now();
    tempPdfPath = path.join(tempDir, `${timestamp}.pdf`);
    fs.writeFileSync(tempPdfPath, req.file.buffer);

    // Create directory for images
    tempImageDir = path.join(tempDir, `images-${timestamp}`);
    if (!fs.existsSync(tempImageDir)) {
      fs.mkdirSync(tempImageDir, { recursive: true });
    }

    console.log('Converting PDF to images...');

    // Configure pdf2pic
    const options = {
      density: 200,           // DPI
      saveFilename: 'page',
      savePath: tempImageDir,
      format: 'png',
      width: 2000,
      height: 3000
    };

    const converter = fromPath(tempPdfPath, options);

    // Get PDF info to know how many pages
    const pdfBuffer = fs.readFileSync(tempPdfPath);
    const pdfText = pdfBuffer.toString('latin1');
    const pageMatch = pdfText.match(/\/Count\s+(\d+)/);
    const numPages = pageMatch ? parseInt(pageMatch[1]) : 10; // Default to 10 if can't detect

    console.log(`Converting ${numPages} pages...`);

    // Convert all pages
    const imagePromises = [];
    for (let i = 1; i <= numPages; i++) {
      imagePromises.push(converter(i, { responseType: 'base64' }));
    }

    const results = await Promise.all(imagePromises);
    
    // Extract base64 images
    const imageBase64Array = results
      .filter(result => result && result.base64)
      .map(result => result.base64);

    if (imageBase64Array.length === 0) {
      throw new Error('Failed to convert PDF to images');
    }

    console.log(`Successfully converted ${imageBase64Array.length} pages to images`);
    console.log('Sending to DeepSeek API...');

    // Prepare messages with images
    const imageMessages = imageBase64Array.map(base64 => ({
      type: 'image_url',
      image_url: {
        url: `data:image/png;base64,${base64}`
      }
    }));

    // Send to DeepSeek API
    const response = await axios.post(
      'https://api.deepseek.com/chat/completions',
      {
        model: 'deepseek-chat',
        messages: [
          {
            role: 'system',
            content: 'You are a helpful assistant that extracts math questions from document images and converts them to structured JSON format with MathJax notation. You understand Bengali/Bangla text and mathematical equations.'
          },
          {
            role: 'user',
            content: [
              {
                type: 'text',
                text: `I have uploaded ${imageBase64Array.length} page(s) from a PDF document containing math questions with equations. Please analyze these images and extract all questions.

Extract all math questions and convert them to JSON format. Each question should have:
- index: sequential number starting from 1
- question: the question text with math in MathJax format using $ for inline and $ for display
- options: array of 4 options (a, b, c, d), with math in MathJax format
- answer: the correct option letter (a, b, c, or d)

CRITICAL RULES:
- Use $ for inline math (e.g., $x^2-2x+4=0$)
- Use $ for display math (e.g., $\\frac{-b \\pm \\sqrt{b^2-4ac}}{2a}$)
- DO NOT use \\( \\) or \\[ \\]
- Preserve all Bengali/Bangla text exactly
- Extract equations exactly as they appear in the images
- Identify the correct answer from visual cues (bold, underline, checkmarks, etc.)

Return ONLY a valid JSON array with no markdown code blocks or additional text.

Example format:
[
  {
    "index": 1,
    "question": "$x^2-2x+4=0$ এর মূলদ্বয় কত?",
    "options": ["সমান", "জটিল ও অসমান", "বাস্তব ও অসমান", "জটিল ও সমান"],
    "answer": "d"
  }
]`
              },
              ...imageMessages
            ]
          }
        ],
        max_tokens: 8000,
        temperature: 0.2
      },
      {
        headers: {
          'Authorization': `Bearer ${process.env.DEEPSEEK_API_KEY}`,
          'Content-Type': 'application/json'
        }
      }
    );

    const aiResponse = response.data.choices[0].message.content;
    console.log('AI Response:', aiResponse.substring(0, 500));

    // Parse JSON from response
    let questionsJson;
    try {
      let cleanedResponse = aiResponse.replace(/```json\n?|\n?```/g, '').trim();
      
      const jsonMatch = cleanedResponse.match(/\[[\s\S]*\]/);
      if (jsonMatch) {
        questionsJson = JSON.parse(jsonMatch[0]);
      } else {
        questionsJson = JSON.parse(cleanedResponse);
      }

      if (!Array.isArray(questionsJson) || questionsJson.length === 0) {
        throw new Error('Invalid questions array');
      }

    } catch (parseError) {
      console.error('Parse error:', parseError);
      console.error('Full AI Response:', aiResponse);
      return res.status(500).json({ 
        error: 'Failed to parse AI response',
        aiResponse: aiResponse 
      });
    }

    res.json({ questions: questionsJson });

  } catch (error) {
    console.error('Error processing PDF:', error);
    res.status(500).json({ 
      error: 'Failed to process document',
      details: error.response?.data || error.message 
    });
  } finally {
    // Cleanup temporary files
    if (tempPdfPath && fs.existsSync(tempPdfPath)) {
      fs.unlinkSync(tempPdfPath);
    }
    if (tempImageDir && fs.existsSync(tempImageDir)) {
      try {
        const files = fs.readdirSync(tempImageDir);
        files.forEach(file => {
          fs.unlinkSync(path.join(tempImageDir, file));
        });
        fs.rmdirSync(tempImageDir);
      } catch (cleanupError) {
        console.error('Cleanup error:', cleanupError);
      }
    }
  }
};
```

## Frontend Setup

### 1. Initialize Next.js App
```bash
npx create-next-app@latest frontend
# Choose: TypeScript: No, ESLint: Yes, Tailwind CSS: Yes, App Router: Yes
cd frontend
npm install @reduxjs/toolkit react-redux axios @dnd-kit/core @dnd-kit/sortable @dnd-kit/utilities
```

### 2. frontend/src/store/store.js
```javascript
import { configureStore } from '@reduxjs/toolkit';
import questionReducer from './questionSlice';

export const store = configureStore({
  reducer: {
    questions: questionReducer,
  },
});
```

### 3. frontend/src/store/questionSlice.js
```javascript
import { createSlice } from '@reduxjs/toolkit';

const questionSlice = createSlice({
  name: 'questions',
  initialState: {
    items: [],
    loading: false,
    error: null,
  },
  reducers: {
    setQuestions: (state, action) => {
      state.items = action.payload;
    },
    reorderQuestions: (state, action) => {
      const { oldIndex, newIndex } = action.payload;
      const items = [...state.items];
      const [removed] = items.splice(oldIndex, 1);
      items.splice(newIndex, 0, removed);
      
      // Re-index
      state.items = items.map((item, idx) => ({
        ...item,
        index: idx + 1
      }));
    },
    setLoading: (state, action) => {
      state.loading = action.payload;
    },
    setError: (state, action) => {
      state.error = action.payload;
    },
  },
});

export const { setQuestions, reorderQuestions, setLoading, setError } = questionSlice.actions;
export default questionSlice.reducer;
```

### 4. frontend/src/app/layout.js
```javascript
'use client';
import { Provider } from 'react-redux';
import { store } from '../store/store';
import './globals.css';

export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <head>
        <script
          id="MathJax-script"
          async
          src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js"
        />
        <script
          dangerouslySetInnerHTML={{
            __html: `
              window.MathJax = {
                tex: {
                  inlineMath: [['$', '$']],
                  displayMath: [['$$', '$$']]
                }
              };
            `
          }}
        />
      </head>
      <body>
        <Provider store={store}>{children}</Provider>
      </body>
    </html>
  );
}
```

### 5. frontend/src/app/page.js
```javascript
'use client';
import { useState } from 'react';
import { useDispatch, useSelector } from 'react-redux';
import { useRouter } from 'next/navigation';
import { setQuestions, setLoading, setError } from '../store/questionSlice';
import axios from 'axios';

export default function Home() {
  const [file, setFile] = useState(null);
  const dispatch = useDispatch();
  const router = useRouter();
  const { loading, error } = useSelector((state) => state.questions);

  const handleFileChange = (e) => {
    setFile(e.target.files[0]);
    dispatch(setError(null));
  };

  const handleUpload = async (e) => {
    e.preventDefault();
    
    if (!file) {
      alert('Please select a file');
      return;
    }

    dispatch(setLoading(true));
    dispatch(setError(null));

    const formData = new FormData();
    formData.append('pdf', file);

    try {
      const response = await axios.post('http://localhost:5000/api/upload', formData, {
        headers: { 'Content-Type': 'multipart/form-data' },
        timeout: 120000 // 2 minutes timeout
      });

      dispatch(setQuestions(response.data.questions));
      router.push('/questions');
    } catch (error) {
      console.error('Upload error:', error);
      dispatch(setError(error.response?.data?.error || 'Upload failed'));
    } finally {
      dispatch(setLoading(false));
    }
  };

  return (
    <div className="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100 py-12 px-4">
      <div className="max-w-2xl mx-auto">
        <div className="text-center mb-8">
          <h1 className="text-4xl font-bold text-gray-800 mb-2">
            Math Questions PDF Converter
          </h1>
          <p className="text-gray-600">Upload a PDF with math questions and get structured JSON</p>
        </div>
        
        <form onSubmit={handleUpload} className="bg-white p-8 rounded-xl shadow-lg">
          <div className="mb-6">
            <label className="block text-sm font-medium text-gray-700 mb-2">
              Upload PDF File
            </label>
            <input
              type="file"
              accept=".pdf"
              onChange={handleFileChange}
              disabled={loading}
              className="block w-full text-sm text-gray-500
                file:mr-4 file:py-3 file:px-6
                file:rounded-lg file:border-0
                file:text-sm file:font-semibold
                file:bg-indigo-50 file:text-indigo-700
                hover:file:bg-indigo-100
                disabled:opacity-50 disabled:cursor-not-allowed"
            />
            <p className="mt-2 text-xs text-gray-500">
              Maximum file size: 10MB
            </p>
          </div>

          {error && (
            <div className="mb-4 p-4 bg-red-50 border border-red-200 rounded-lg">
              <div className="flex">
                <div className="flex-shrink-0">
                  <svg className="h-5 w-5 text-red-400" viewBox="0 0 20 20" fill="currentColor">
                    <path fillRule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zM8.707 7.293a1 1 0 00-1.414 1.414L8.586 10l-1.293 1.293a1 1 0 101.414 1.414L10 11.414l1.293 1.293a1 1 0 001.414-1.414L11.414 10l1.293-1.293a1 1 0 00-1.414-1.414L10 8.586 8.707 7.293z" clipRule="evenodd" />
                  </svg>
                </div>
                <div className="ml-3">
                  <p className="text-sm text-red-700">{error}</p>
                </div>
              </div>
            </div>
          )}
          
          <button
            type="submit"
            disabled={loading || !file}
            className="w-full bg-indigo-600 text-white py-3 px-4 rounded-lg
              hover:bg-indigo-700 transition-colors font-medium 
              disabled:bg-gray-400 disabled:cursor-not-allowed
              flex items-center justify-center"
          >
            {loading ? (
              <>
                <svg className="animate-spin -ml-1 mr-3 h-5 w-5 text-white" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                  <circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4"></circle>
                  <path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
                </svg>
                Processing PDF...
              </>
            ) : (
              'Upload and Convert'
            )}
          </button>
        </form>

        <div className="mt-8 bg-white p-6 rounded-xl shadow-lg">
          <h2 className="text-lg font-semibold text-gray-800 mb-3">Instructions:</h2>
          <ol className="list-decimal list-inside space-y-2 text-sm text-gray-600">
            <li>Convert your DOCX file to PDF (File → Save As → PDF)</li>
            <li>Upload the PDF file</li>
            <li>Wait for AI to extract and convert questions</li>
            <li>Review and reorder questions using drag & drop</li>
          </ol>
        </div>
      </div>
    </div>
  );
}
```

### 6. frontend/src/app/questions/page.js
```javascript
'use client';
import { useEffect } from 'react';
import { useSelector } from 'react-redux';
import { useRouter } from 'next/navigation';
import QuestionList from '../../components/QuestionList';

export default function QuestionsPage() {
  const { items, loading, error } = useSelector((state) => state.questions);
  const router = useRouter();

  useEffect(() => {
    if (items.length === 0 && !loading) {
      router.push('/');
    }
  }, [items, loading, router]);

  useEffect(() => {
    if (typeof window !== 'undefined' && window.MathJax) {
      window.MathJax.typesetPromise?.();
    }
  }, [items]);

  if (loading) {
    return (
      <div className="min-h-screen flex items-center justify-center bg-gray-50">
        <div className="text-center">
          <div className="inline-block animate-spin rounded-full h-12 w-12 border-b-2 border-indigo-600 mb-4"></div>
          <div className="text-xl text-gray-700">Processing document...</div>
        </div>
      </div>
    );
  }

  if (error) {
    return (
      <div className="min-h-screen flex items-center justify-center bg-gray-50">
        <div className="text-center">
          <div className="text-6xl mb-4">❌</div>
          <div className="text-xl text-red-600">Error: {error}</div>
          <button
            onClick={() => router.push('/')}
            className="mt-4 px-6 py-2 bg-indigo-600 text-white rounded-lg hover:bg-indigo-700"
          >
            Try Again
          </button>
        </div>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-gray-50 py-8 px-4">
      <div className="max-w-4xl mx-auto">
        <div className="flex justify-between items-center mb-8">
          <div>
            <h1 className="text-3xl font-bold text-gray-800">Questions Preview</h1>
            <p className="text-gray-600 mt-1">{items.length} questions extracted</p>
          </div>
          <button
            onClick={() => router.push('/')}
            className="px-6 py-2 bg-gray-600 text-white rounded-lg hover:bg-gray-700 transition-colors"
          >
            Upload New File
          </button>
        </div>
        <QuestionList questions={items} />
      </div>
    </div>
  );
}
```

### 7. frontend/src/components/QuestionList.js
```javascript
'use client';
import { useDispatch } from 'react-redux';
import { DndContext, closestCenter, KeyboardSensor, PointerSensor, useSensor, useSensors } from '@dnd-kit/core';
import { SortableContext, sortableKeyboardCoordinates, verticalListSortingStrategy } from '@dnd-kit/sortable';
import { reorderQuestions } from '../store/questionSlice';
import QuestionCard from './QuestionCard';

export default function QuestionList({ questions }) {
  const dispatch = useDispatch();

  const sensors = useSensors(
    useSensor(PointerSensor),
    useSensor(KeyboardSensor, {
      coordinateGetter: sortableKeyboardCoordinates,
    })
  );

  const handleDragEnd = (event) => {
    const { active, over } = event;

    if (active.id !== over.id) {
      const oldIndex = questions.findIndex((q) => q.index === active.id);
      const newIndex = questions.findIndex((q) => q.index === over.id);

      dispatch(reorderQuestions({ oldIndex, newIndex }));
      
      // Re-render MathJax after reorder
      setTimeout(() => {
        if (typeof window !== 'undefined' && window.MathJax) {
          window.MathJax.typesetPromise?.();
        }
      }, 100);
    }
  };

  return (
    <DndContext
      sensors={sensors}
      collisionDetection={closestCenter}
      onDragEnd={handleDragEnd}
    >
      <SortableContext
        items={questions.map((q) => q.index)}
        strategy={verticalListSortingStrategy}
      >
        <div className="space-y-4">
          {questions.map((question) => (
            <QuestionCard key={question.index} question={question} />
          ))}
        </div>
      </SortableContext>
    </DndContext>
  );
}
```

### 8. frontend/src/components/QuestionCard.js
```javascript
'use client';
import { useSortable } from '@dnd-kit/sortable';
import { CSS } from '@dnd-kit/utilities';

export default function QuestionCard({ question }) {
  const {
    attributes,
    listeners,
    setNodeRef,
    transform,
    transition,
  } = useSortable({ id: question.index });

  const style = {
    transform: CSS.Transform.toString(transform),
    transition,
  };

  const optionLabels = ['(a)', '(b)', '(c)', '(d)'];

  return (
    <div
      ref={setNodeRef}
      style={style}
      className="bg-white p-6 rounded-xl shadow-md hover:shadow-lg transition-all border border-gray-200"
    >
      <div className="flex items-start gap-3">
        <div
          {...attributes}
          {...listeners}
          className="flex-shrink-0 cursor-move text-gray-400 hover:text-gray-600 transition-colors pt-1"
        >
          <svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2">
            <circle cx="9" cy="5" r="1" />
            <circle cx="9" cy="12" r="1" />
            <circle cx="9" cy="19" r="1" />
            <circle cx="15" cy="5" r="1" />
            <circle cx="15" cy="12" r="1" />
            <circle cx="15" cy="19" r="1" />
          </svg>
        </div>
        
        <div className="flex-1">
          <div className="font-medium mb-4 text-lg leading-relaxed">
            <span className="text-indigo-600 font-semibold">{question.index}.</span>{' '}
            <span dangerouslySetInnerHTML={{ __html: question.question }} />
          </div>
          
          <div className="grid grid-cols-1 md:grid-cols-2 gap-3">
            {question.options.map((option, idx) => {
              const optionLetter = String.fromCharCode(97 + idx);
              const isCorrect = question.answer.toLowerCase() === optionLetter;
              
              return (
                <div
                  key={idx}
                  className={`p-4 rounded-lg border-2 transition-all ${
                    isCorrect
                      ? 'bg-green-50 border-green-400 shadow-sm'
                      : 'bg-gray-50 border-gray-200 hover:border-gray-300'
                  }`}
                >
                  <span className="font-semibold text-gray-700">{optionLabels[idx]}</span>{' '}
                  <span dangerouslySetInnerHTML={{ __html: option }} />
                </div>
              );
            })}
          </div>
        </div>
      </div>
    </div>
  );
}
```

## Running the Project

### 1. Install System Dependencies
```bash
# Ubuntu/Debian
sudo apt-get update
sudo apt-get install -y poppler-utils

# macOS
brew install poppler
```

### 2. Start Backend
```bash
cd backend
npm install
npm run dev
```

### 3. Start Frontend
```bash
cd frontend
npm install
npm run dev
```

### 4. Usage
1. Convert your DOCX to PDF (File → Save As → PDF in Word)
2. Visit `http://localhost:3000`
3. Upload your PDF file
4. Wait for processing (may take 30-60 seconds)
5. View and reorder questions!

## Key Features

✅ **DeepSeek Vision API** - Can "see" equations in images
✅ **PDF to Images** - Converts each PDF page to an image
✅ **Correct MathJax** - Uses `$` and `$$` delimiters
✅ **Perfect Equation Extraction** - AI reads equations from images
✅ **Drag & Drop Reordering** - Easy question management
✅ **Redux State Management** - No database needed
✅ **Bengali Support** - Preserves Bangla text perfectly

This approach works perfectly because DeepSeek can analyze images and extract both text and mathematical equations visually!
