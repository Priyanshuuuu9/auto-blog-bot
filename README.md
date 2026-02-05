# auto-blog-bot
const express = require('express');
const OpenAI = require('openai');
const Parser = require('rss-parser');
const { google } = require('googleapis');

const app = express();
const parser = new Parser();

// Environment Variables
const OPENAI_API_KEY = process.env.OPENAI_API_KEY;
const GOOGLE_CLIENT_ID = process.env.GOOGLE_CLIENT_ID;
const GOOGLE_CLIENT_SECRET = process.env.GOOGLE_CLIENT_SECRET;
const GOOGLE_REFRESH_TOKEN = process.env.GOOGLE_REFRESH_TOKEN;
const BLOGGER_BLOG_ID = process.env.BLOGGER_BLOG_ID;

// OpenAI Setup
const openai = new OpenAI({ apiKey: OPENAI_API_KEY });

// Google OAuth Setup
const oauth2Client = new google.auth.OAuth2(
  GOOGLE_CLIENT_ID,
  GOOGLE_CLIENT_SECRET
);
oauth2Client.setCredentials({ refresh_token: GOOGLE_REFRESH_TOKEN });

const blogger = google.blogger({ version: 'v3', auth: oauth2Client });

// RSS Feeds
const RSS_FEEDS = [
  { name: 'Smartprix', url: 'https://www.smartprix.com/bytes/feed/' },
  { name: '91mobiles', url: 'https://www.91mobiles.com/hub/feed/' },
  { name: 'Gadgets360', url: 'https://feeds.feedburner.com/gadgets360-latest' }
];

// Store published articles
let publishedUrls = [];

// Fetch RSS Articles
async function fetchArticles() {
  console.log('🔍 Scanning for new articles...');
  let articles = [];
  
  for (const feed of RSS_FEEDS) {
    try {
      const result = await parser.parseURL(feed.url);
      const newArticles = result.items.slice(0, 3).map(item => ({
        title: item.title,
        link: item.link,
        source: feed.name,
        pubDate: item.pubDate,
        content: item.contentSnippet || item.content || ''
      }));
      articles = [...articles, ...newArticles];
      console.log(`✅ ${feed.name}: ${newArticles.length} articles found`);
    } catch (error) {
      console.log(`❌ ${feed.name}: Error fetching`);
    }
  }
  
  return articles.filter(a => !publishedUrls.includes(a.link));
}

// Rewrite with AI
async function rewriteArticle(article) {
  console.log(`✍️ Rewriting: ${article.title}`);
  
  const prompt = `You are a professional tech blogger. Rewrite this article in a completely unique way:

Title: ${article.title}
Content: ${article.content}

Requirements:
1. Create a new SEO-optimized title
2. Write 1500+ words unique content
3. Use simple Hindi-English mixed language
4. Add proper headings (H2, H3)
5. Make it engaging and informative
6. Include a meta description
7. Suggest 5 SEO keywords

Format your response as JSON:
{
  "title": "New Title",
  "metaDescription": "Meta description here",
  "keywords": ["keyword1", "keyword2"],
  "content": "<h2>Heading</h2><p>Content...</p>"
}`;

  try {
    const response = await openai.chat.completions.create({
      model: 'gpt-4o-mini',
      messages: [{ role: 'user', content: prompt }],
      max_tokens: 4000
    });
    
    const result = JSON.parse(response.choices[0].message.content);
    console.log(`✅ Rewritten: ${result.title}`);
    return result;
  } catch (error) {
    console.log('❌ AI Error:', error.message);
    return null;
  }
}

// Publish to Blogger
async function publishToBlogger(article) {
  console.log(`🚀 Publishing: ${article.title}`);
  
  try {
    const response = await blogger.posts.insert({
      blogId: BLOGGER_BLOG_ID,
      requestBody: {
        title: article.title,
        content: article.content,
        labels: article.keywords
      }
    });
    
    console.log(`✅ Published: ${response.data.url}`);
    return response.data;
  } catch (error) {
    console.log('❌ Publish Error:', error.message);
    return null;
  }
}

// Main Auto-Blog Function
async function autoBlog() {
  console.log('\n========== AUTO-BLOG CYCLE START ==========\n');
  
  // Step 1: Fetch new articles
  const articles = await fetchArticles();
  console.log(`📰 Found ${articles.length} new articles\n`);
  
  if (articles.length === 0) {
    console.log('😴 No new articles. Waiting...\n');
    return;
  }
  
  // Step 2: Process each article
  for (const article of articles.slice(0, 3)) {
    // Rewrite
    const rewritten = await rewriteArticle(article);
    if (!rewritten) continue;
    
    // Publish
    const published = await publishToBlogger(rewritten);
    if (published) {
      publishedUrls.push(article.link);
    }
    
    // Wait 30 seconds between posts
    await new Promise(r => setTimeout(r, 30000));
  }
  
  console.log('\n========== CYCLE COMPLETE ==========\n');
}

// API Endpoints
app.get('/', (req, res) => {
  res.json({
    status: '🟢 AutoBlog Pro Running',
    published: publishedUrls.length,
    lastRun: new Date().toISOString()
  });
});

app.get('/run', async (req, res) => {
  autoBlog();
  res.json({ message: '🚀 Auto-blog cycle started!' });
});

// Start Server
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`🚀 Server running on port ${PORT}`);
  
  // Run every 30 minutes
  setInterval(autoBlog, 30 * 60 * 1000);
  
  // Run immediately on start
  setTimeout(autoBlog, 5000);
});
