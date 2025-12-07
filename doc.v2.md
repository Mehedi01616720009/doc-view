# Math Questions DOCX Converter

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
npm install express cors dotenv multer axios form-data
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
    "axios": "^1.6.2",
    "form-data": "^4.0.0"
  }
}
```

### 3. backend/.env
```
PORT=5000
DEEPSEEK_API_KEY=your_deepseek_api_key_here
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
import axios from 'axios';

export const processDocx = async (req, res) => {
  try {
    if (!req.file) {
      return res.status(400).json({ error: 'No file uploaded' });
    }

    const fileBuffer = req.file.buffer;
    const base64File = fileBuffer.toString('base64');

    // Send to DeepSeek API
    const response = await axios.post(
      'https://api.deepseek.com/chat/completions',
      {
        model: 'deepseek-chat',
        messages: [
          {
            role: 'system',
            content: 'You are a helpful assistant that extracts math questions from documents and converts them to structured JSON format with MathJax notation.'
          },
          {
            role: 'user',
            content: [
              {
                type: 'file',
                file_url: {
                  url: `data:application/vnd.openxmlformats-officedocument.wordprocessingml.document;base64,${base64File}`
                }
              },
              {
                type: 'text',
                text: `Extract all math questions from this document and convert them to JSON format. Each question should have:
- index: sequential number starting from 1
- question: the question text converted to MathJax format (use \\( \\) for inline math and \\[ \\] for display math)
- options: array of 4 options, each converted to MathJax format
- answer: the correct option letter (a, b, c, or d)

Return ONLY a valid JSON array of questions, nothing else. Example format:
[
  {
    "index": 1,
    "question": "\\(x^2-2x+4=0\\) এর মূলদ্বয় কত?",
    "options": ["সমান", "জটিল ও অসমান", "বাস্তব ও অসমান", "জটিল ও সমান"],
    "answer": "d"
  }
]`
              }
            ]
          }
        ],
        max_tokens: 4000,
        temperature: 0.3
      },
      {
        headers: {
          'Authorization': `Bearer ${process.env.DEEPSEEK_API_KEY}`,
          'Content-Type': 'application/json'
        }
      }
    );

    const aiResponse = response.data.choices[0].message.content;
    
    // Extract JSON from response
    let questionsJson;
    try {
      // Remove markdown code blocks if present
      let cleanedResponse = aiResponse.replace(/```json\n?|\n?```/g, '').trim();
      
      const jsonMatch = cleanedResponse.match(/\[[\s\S]*\]/);
      if (jsonMatch) {
        questionsJson = JSON.parse(jsonMatch[0]);
      } else {
        questionsJson = JSON.parse(cleanedResponse);
      }
    } catch (parseError) {
      console.error('Parse error:', parseError);
      console.error('AI Response:', aiResponse);
      return res.status(500).json({ 
        error: 'Failed to parse AI response',
        aiResponse: aiResponse 
      });
    }

    res.json({ questions: questionsJson });

  } catch (error) {
    console.error('Error processing DOCX:', error.response?.data || error.message);
    res.status(500).json({ 
      error: 'Failed to process document',
      details: error.response?.data || error.message 
    });
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
                  inlineMath: [['\\\\(', '\\\\)']],
                  displayMath: [['\\\\[', '\\\\]']]
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
import { useDispatch } from 'react-redux';
import { useRouter } from 'next/navigation';
import { setQuestions, setLoading, setError } from '../store/questionSlice';
import axios from 'axios';

export default function Home() {
  const [file, setFile] = useState(null);
  const dispatch = useDispatch();
  const router = useRouter();

  const handleFileChange = (e) => {
    setFile(e.target.files[0]);
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
          
          <button
            type="submit"
            className="w-full bg-blue-600 text-white py-2 px-4 rounded-md
              hover:bg-blue-700 transition-colors font-medium"
          >
            Upload and Convert
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
import QuestionList from '../../components/QuestionList';

export default function QuestionsPage() {
  const { items, loading, error } = useSelector((state) => state.questions);

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
        <h1 className="text-3xl font-bold mb-8">Questions Preview</h1>
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
      className="bg-white p-6 rounded-lg shadow-md cursor-move hover:shadow-lg transition-shadow"
    >
      <div {...attributes} {...listeners} className="mb-4">
        <div className="flex items-start gap-2">
          <span className="text-gray-500 font-mono text-sm">≡</span>
          <div className="flex-1">
            <div className="font-medium mb-3 text-lg">
              {question.index}. <span dangerouslySetInnerHTML={{ __html: question.question }} />
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

## Notes
- DeepSeek API accepts DOCX files directly as base64
- Questions are automatically converted to MathJax format
- Drag and drop any question to reorder
- Correct answers are highlighted in green
- Questions are stored in Redux (no database needed)
