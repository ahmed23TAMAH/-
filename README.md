import React, { useState, useRef, useEffect } from 'react';
import { Message } from './types';
import { streamGeminiResponse } from './services/geminiService';
import { ChatMessage } from './components/ChatMessage';
import { QuickActions } from './components/QuickActions';

const App: React.FC = () => {
  const [messages, setMessages] = useState<Message[]>([
    {
      role: 'model',
      content: 'أهلاً وسهلاً',
    },
  ]);
  const [input, setInput] = useState('');
  const [isLoading, setIsLoading] = useState(false);
  const messagesEndRef = useRef<HTMLDivElement>(null);

  const scrollToBottom = () => {
    messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' });
  };

  useEffect(() => {
    scrollToBottom();
  }, [messages]);

  const handleSend = async (text: string) => {
    if (!text.trim() || isLoading) return;

    const userMessage: Message = { role: 'user', content: text };
    setMessages((prev) => [...prev, userMessage]);
    setInput('');
    setIsLoading(true);

    const modelMessageId = Date.now();
    setMessages((prev) => [
      ...prev,
      { role: 'model', content: '' },
    ]);

    let fullResponse = '';

    try {
      await streamGeminiResponse(
        [...messages, userMessage], 
        text,
        (chunk) => {
            fullResponse += chunk;
            setMessages((prev) => {
              const newMessages = [...prev];
              const lastMsgIndex = newMessages.length - 1;
              if (newMessages[lastMsgIndex].role === 'model') {
                 newMessages[lastMsgIndex] = { 
                   ...newMessages[lastMsgIndex], 
                   content: fullResponse 
                 };
              }
              return newMessages;
            });
        }
      );
    } catch (error) {
      console.error(error);
      setMessages((prev) => {
        const newMessages = [...prev];
        const lastMsgIndex = newMessages.length - 1;
        if (newMessages[lastMsgIndex].role === 'model' && !newMessages[lastMsgIndex].content) {
             newMessages[lastMsgIndex] = { 
                 role: 'model', 
                 content: 'عذراً، حدث خطأ في الاتصال. حاول مرة أخرى.', 
                 isError: true 
             };
        }
        return newMessages;
      });
    } finally {
      setIsLoading(false);
    }
  };

  const handleKeyPress = (e: React.KeyboardEvent) => {
    if (e.key === 'Enter' && !e.shiftKey) {
      e.preventDefault();
      handleSend(input);
    }
  };

  return (
    <div className="flex flex-col h-screen bg-slate-900 text-slate-100 font-sans selection:bg-orange-500/30">
      {/* Premium Header */}
      <header className="sticky top-0 z-20 px-4 py-3 bg-slate-900/80 backdrop-blur-md border-b border-white/5 shadow-2xl">
        <div className="max-w-3xl mx-auto flex items-center justify-between">
          <div className="flex items-center gap-4">
            {/* Logo */}
            <div className="relative group cursor-default">
               <div className="absolute -inset-0.5 bg-gradient-to-r from-orange-500 to-blue-600 rounded-xl blur opacity-30 group-hover:opacity-60 transition duration-500"></div>
               <div className="relative w-11 h-11 bg-slate-800 rounded-lg flex items-center justify-center border border-slate-700/50 shadow-inner">
                 <span className="text-2xl font-black text-transparent bg-clip-text bg-gradient-to-br from-orange-400 to-blue-500 select-none">R</span>
               </div>
            </div>
            
            {/* Title */}
            <div className="flex flex-col justify-center">
              <h1 className="text-xl md:text-2xl font-black tracking-tighter text-white leading-none flex items-center gap-1">
                RIVALS <span className="text-orange-500">مارفل</span>
              </h1>
            </div>
          </div>

          {/* Online Indicator */}
          <div className="hidden sm:flex items-center gap-2 px-3 py-1.5 bg-slate-800/50 rounded-full border border-slate-700/50 backdrop-blur-sm">
             <span className="relative flex h-2 w-2">
               <span className="animate-ping absolute inline-flex h-full w-full rounded-full bg-green-400 opacity-75"></span>
               <span className="relative inline-flex rounded-full h-2 w-2 bg-green-500"></span>
             </span>
             <span className="text-[10px] font-bold text-slate-400 uppercase tracking-wider">System Online</span>
          </div>
        </div>
      </header>

      {/* Chat Area */}
      <div className="flex-1 overflow-y-auto p-4 md:p-6 custom-scrollbar bg-gradient-to-b from-slate-900 via-slate-900 to-slate-900/50">
        <div className="max-w-3xl mx-auto flex flex-col min-h-full justify-end">
           {messages.map((msg, idx) => (
             <ChatMessage key={idx} message={msg} />
           ))}
           
           {isLoading && messages[messages.length - 1].content === '' && (
             <div className="flex justify-start mb-4">
               <div className="bg-slate-800/80 backdrop-blur-sm rounded-2xl rounded-bl-none px-4 py-3 border border-slate-700/50">
                 <div className="flex gap-1.5 items-center h-4">
                   <div className="w-1.5 h-1.5 bg-blue-400 rounded-full animate-bounce"></div>
                   <div className="w-1.5 h-1.5 bg-blue-400 rounded-full animate-bounce delay-75"></div>
                   <div className="w-1.5 h-1.5 bg-blue-400 rounded-full animate-bounce delay-150"></div>
                 </div>
               </div>
             </div>
           )}
           <div ref={messagesEndRef} />
        </div>
      </div>

      {/* Input Area */}
      <div className="p-4 bg-slate-900/95 border-t border-white/5 backdrop-blur-sm z-10">
        <div className="max-w-3xl mx-auto">
          <QuickActions onSelect={(prompt) => handleSend(prompt)} disabled={isLoading} />
          
          <div className="relative group flex items-end gap-2 bg-slate-800/50 p-2 rounded-2xl border border-slate-700/50 focus-within:border-blue-500/50 focus-within:bg-slate-800 focus-within:shadow-lg focus-within:shadow-blue-500/5 transition-all duration-300">
            <textarea
              value={input}
              onChange={(e) => setInput(e.target.value)}
              onKeyDown={handleKeyPress}
              placeholder="اكتب سؤالك هنا يا بطل..."
              className="w-full bg-transparent border-none focus:ring-0 resize-none max-h-32 py-3 px-3 text-slate-100 placeholder-slate-500 text-sm md:text-base font-medium"
              rows={1}
              style={{ minHeight: '48px' }}
              disabled={isLoading}
            />
            <button
              onClick={() => handleSend(input)}
              disabled={!input.trim() || isLoading}
              className="p-3 bg-gradient-to-tr from-blue-600 to-blue-500 hover:from-blue-500 hover:to-blue-400 disabled:from-slate-700 disabled:to-slate-700 text-white rounded-xl transition-all flex-shrink-0 mb-0.5 shadow-lg shadow-blue-500/20 hover:shadow-blue-500/30 active:scale-95 disabled:shadow-none"
            >
              <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="currentColor" className="w-5 h-5 rtl:-scale-x-100">
                <path d="M3.478 2.405a.75.75 0 00-.926.94l2.432 7.905H13.5a.75.75 0 010 1.5H4.984l-2.432 7.905a.75.75 0 00.926.94 60.519 60.519 0 0018.445-8.986.75.75 0 000-1.218A60.517 60.517 0 003.478 2.405z" />
              </svg>
            </button>
          </div>
          
          <div className="text-center mt-4 flex flex-col items-center justify-center gap-2 opacity-80 hover:opacity-100 transition-opacity duration-300">
             <div className="flex items-center gap-2">
                <div className="h-px w-8 bg-slate-700"></div>
                <p className="text-[10px] text-slate-400 font-bold tracking-wider uppercase">
                  صنع من أحمد <span className="text-red-500 animate-pulse">❤️</span>
                </p>
                <div className="h-px w-8 bg-slate-700"></div>
             </div>
             {/* Social Links */}
             <div className="flex gap-4 mt-1">
                <a href="https://x.com/ad_2h2?s=21" target="_blank" rel="noopener noreferrer" className="text-slate-500 hover:text-blue-400 transition-colors">
                  <svg className="w-5 h-5" fill="currentColor" viewBox="0 0 24 24" aria-hidden="true">
                    <path d="M8.29 20.251c7.547 0 11.675-6.253 11.675-11.675 0-.178 0-.355-.012-.53A8.348 8.348 0 0022 5.92a8.19 8.19 0 01-2.357.646 4.118 4.118 0 001.804-2.27 8.224 8.224 0 01-2.605.996 4.107 4.107 0 00-6.993 3.743 11.65 11.65 0 01-8.457-4.287 4.106 4.106 0 001.27 5.477A4.072 4.072 0 012.8 9.713v.052a4.105 4.105 0 003.292 4.022 4.095 4.095 0 01-1.853.07 4.108 4.108 0 003.834 2.85A8.233 8.233 0 012 18.407a11.616 11.616 0 006.29 1.84" />
                  </svg>
                </a>
             </div>
          </div>
        </div>
      </div>
    </div>
  );
};

export default App; 
