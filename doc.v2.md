# Math Questions DOCX Converter (OpenAI Version)

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
│   │   │   ├── UploadForm.js
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

### 1. Initialize Backend
```bash
mkdir backend && cd backend
npm init -y
npm install express cors dotenv multer openai
```

### 2. backend/package.json
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
    "openai": "^4.20.0"
  }
}
```

### 3. backend/.env
```
PORT=5000
OPENAI_API_KEY=your_openai_api_key_here
```

### 4. backend/server.js
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

### 5. backend/routes/upload.js
```javascript
import express from 'express';
import multer from 'multer';
import { processDocx } from '../controllers/questionController.js';

const router = express.Router();

const storage = multer.memoryStorage();
const upload = multer({ 
  storage,
  fileFilter: (req, file, cb) => {
    if (file.mimetype === 'application/vnd.openxmlformats-officedocument.wordprocessingml.document') {
      cb(null, true);
    } else {
      cb(new Error('Only .docx files are allowed'));
    }
  }
});

router.post('/upload', upload.single('docx'), processDocx);

export default router;
```

### 6. backend/controllers/questionController.js
```javascript
import OpenAI from 'openai';
import fs from 'fs';
import path from 'path';
import { fileURLToPath } from 'url';

const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);

const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY
});

export const processDocx = async (req, res) => {
  let tempFilePath = null;

  try {
    if (!req.file) {
      return res.status(400).json({ error: 'No file uploaded' });
    }

    // Save buffer to temporary file (OpenAI needs file path)
    const tempDir = path.join(__dirname, '../temp');
    if (!fs.existsSync(tempDir)) {
      fs.mkdirSync(tempDir, { recursive: true });
    }

    tempFilePath = path.join(tempDir, `${Date.now()}-${req.file.originalname}`);
    fs.writeFileSync(tempFilePath, req.file.buffer);

    // Upload file to OpenAI
    const file = await openai.files.create({
      file: fs.createReadStream(tempFilePath),
      purpose: 'assistants'
    });

    console.log('File uploaded to OpenAI:', file.id);

    // Create assistant with file
    const assistant = await openai.beta.assistants.create({
      name: "Math Questions Extractor",
      instructions: `You are a helpful assistant that extracts math questions from DOCX documents and converts them to structured JSON format with MathJax notation. 

CRITICAL: Use proper MathJax delimiters:
- For inline math: $x^2$ (single dollar signs)
- For display math: $$\\frac{a}{b}$$ (double dollar signs)
- DO NOT use \\( \\) or \\[ \\]

Preserve Bengali/Bangla text exactly as written.`,
      model: "gpt-4-turbo-preview",
      tools: [{ type: "retrieval" }],
      file_ids: [file.id]
    });

    // Create thread
    const thread = await openai.beta.threads.create();

    // Send message
    await openai.beta.threads.messages.create(thread.id, {
      role: "user",
      content: `Extract all math questions from the uploaded DOCX file and convert them to JSON format.

Each question should have:
- index: sequential number starting from 1
- question: the question text with math in MathJax format using $ for inline and $$ for display
- options: array of 4 options (a, b, c, d), with math in MathJax format
- answer: the correct option letter (a, b, c, or d)

IMPORTANT RULES:
- Use $ for inline math (e.g., $x^2-2x+4=0$)
- Use $$ for display math (e.g., $$\\frac{-b \\pm \\sqrt{b^2-4ac}}{2a}$$)
- Preserve all Bengali/Bangla text exactly
- Extract equations exactly as they appear in the document
- Identify the correct answer from the document

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
    });

    // Run assistant
    const run = await openai.beta.threads.runs.create(thread.id, {
      assistant_id: assistant.id
    });

    // Poll for completion
    let runStatus = await openai.beta.threads.runs.retrieve(thread.id, run.id);
    let attempts = 0;
    const maxAttempts = 60; // 60 seconds timeout

    while (runStatus.status !== 'completed' && attempts < maxAttempts) {
      await new Promise(resolve => setTimeout(resolve, 1000));
      runStatus = await openai.beta.threads.runs.retrieve(thread.id, run.id);
      attempts++;

      if (runStatus.status === 'failed' || runStatus.status === 'cancelled') {
        throw new Error(`Assistant run ${runStatus.status}`);
      }
    }

    if (runStatus.status !== 'completed') {
      throw new Error('Assistant run timed out');
    }

    // Get messages
    const messages = await openai.beta.threads.messages.list(thread.id);
    const assistantMessage = messages.data.find(m => m.role === 'assistant');
    
    if (!assistantMessage || !assistantMessage.content[0]) {
      throw new Error('No response from assistant');
    }

    const aiResponse = assistantMessage.content[0].text.value;
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

    // Cleanup
    await openai.beta.assistants.del(assistant.id);
    await openai.files.del(file.id);

    res.json({ questions: questionsJson });

  } catch (error) {
    console.error('Error processing DOCX:', error);
    res.status(500).json({ 
      error: 'Failed to process document',
      details: error.message 
    });
  } finally {
    // Delete temporary file
    if (tempFilePath && fs.existsSync(tempFilePath)) {
      fs.unlinkSync(tempFilePath);
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
    formData.append('docx', file);

    try {
      const response = await axios.post('http://localhost:5000/api/upload', formData, {
        headers: { 'Content-Type': 'multipart/form-data' }
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
    <div className="min-h-screen bg-gray-50 py-12 px-4">
      <div className="max-w-2xl mx-auto">
        <h1 className="text-3xl font-bold text-center mb-8">
          Math Questions DOCX Converter
        </h1>
        
        <form onSubmit={handleUpload} className="bg-white p-8 rounded-lg shadow-md">
          <div className="mb-6">
            <label className="block text-sm font-medium text-gray-700 mb-2">
              Upload DOCX File
            </label>
            <input
              type="file"
              accept=".docx"
              onChange={handleFileChange}
              className="block w-full text-sm text-gray-500
                file:mr-4 file:py-2 file:px-4
                file:rounded-md file:border-0
                file:text-sm file:font-semibold
                file:bg-blue-50 file:text-blue-700
                hover:file:bg-blue-100"
            />
          </div>

          {error && (
            <div className="mb-4 p-3 bg-red-50 border border-red-200 rounded text-red-700 text-sm">
              {error}
            </div>
          )}
          
          <button
            type="submit"
            disabled={loading}
            className="w-full bg-blue-600 text-white py-2 px-4 rounded-md
              hover:bg-blue-700 transition-colors font-medium disabled:bg-gray-400"
          >
            {loading ? 'Processing...' : 'Upload and Convert'}
          </button>
        </form>
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
      <div className="min-h-screen flex items-center justify-center">
        <div className="text-xl">Processing document...</div>
      </div>
    );
  }

  if (error) {
    return (
      <div className="min-h-screen flex items-center justify-center">
        <div className="text-xl text-red-600">Error: {error}</div>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-gray-50 py-8 px-4">
      <div className="max-w-4xl mx-auto">
        <div className="flex justify-between items-center mb-8">
          <h1 className="text-3xl font-bold">Questions Preview</h1>
          <button
            onClick={() => router.push('/')}
            className="px-4 py-2 bg-gray-600 text-white rounded hover:bg-gray-700"
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
import { arrayMove, SortableContext, sortableKeyboardCoordinates, verticalListSortingStrategy } from '@dnd-kit/sortable';
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
      className="bg-white p-6 rounded-lg shadow-md hover:shadow-lg transition-shadow"
    >
      <div className="mb-4">
        <div className="flex items-start gap-3">
          <span 
            {...attributes} 
            {...listeners}
            className="text-gray-400 cursor-move hover:text-gray-600 text-xl mt-1"
          >
            ⋮⋮
          </span>
          <div className="flex-1">
            <div className="font-medium mb-3 text-lg">
              <span className="text-gray-600">{question.index}.</span>{' '}
              <span dangerouslySetInnerHTML={{ __html: question.question }} />
            </div>
            
            <div className="grid grid-cols-1 md:grid-cols-2 gap-2">
              {question.options.map((option, idx) => {
                const optionLetter = String.fromCharCode(97 + idx);
                const isCorrect = question.answer.toLowerCase() === optionLetter;
                
                return (
                  <div
                    key={idx}
                    className={`p-3 rounded border ${
                      isCorrect
                        ? 'bg-green-50 border-green-300'
                        : 'bg-gray-50 border-gray-200'
                    }`}
                  >
                    <span className="font-medium">{optionLabels[idx]}</span>{' '}
                    <span dangerouslySetInnerHTML={{ __html: option }} />
                  </div>
                );
              })}
            </div>
          </div>
        </div>
      </div>
    </div>
  );
}
```

## Running the Project

### 1. Start Backend
```bash
cd backend
npm run dev
```

### 2. Start Frontend
```bash
cd frontend
npm run dev
```

Visit `http://localhost:3000` to upload your DOCX file!

## Key Features

✅ **OpenAI API** - Supports DOCX files with equations directly
✅ **Correct MathJax** - Uses `$` for inline and `$$` for display math
✅ **No Mammoth** - Direct DOCX processing by OpenAI
✅ **Equation Extraction** - Preserves exact equations from Word documents
✅ **Drag & Drop** - Reorder questions with automatic re-indexing
✅ **Redux State** - No database needed
✅ **Bengali Support** - Preserves Bangla text perfectly

The OpenAI Assistants API can read DOCX files directly and extract equations properly!
