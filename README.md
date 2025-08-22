this is just a prototype of a example of what you can make in python. this is a search engine that uses tkinter and python aswell as bing to form a search engine thats minimal and working. the rest search engines like anything other than bing is not working. heres  the code:

import threading
import webbrowser
import logging
import time
import random
from urllib.parse import urlparse, quote_plus, urlencode
import tkinter as tk
from tkinter import ttk, messagebox
from playwright.sync_api import sync_playwright, TimeoutError as PlaywrightTimeout
import sys
import traceback
import json

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('web_search.log'),
        logging.StreamHandler(sys.stdout)
    ]
)
logger = logging.getLogger(__name__)

RESULTS_PER_PAGE = 10
DEFAULT_TIMEOUT = 15000  # 15 seconds

class SearchError(Exception):
    """Custom exception for search-related errors"""
    pass

def get_random_user_agent():
    """Return a random realistic user agent"""
    user_agents = [
        'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/122.0.0.0 Safari/537.36',
        'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/121.0.0.0 Safari/537.36',
        'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36',
        'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/122.0.0.0 Safari/537.36',
        'Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/122.0.0.0 Safari/537.36'
    ]
    return random.choice(user_agents)

def search_bing(query, page_num=1, timeout=DEFAULT_TIMEOUT):
    """Search using Bing (more bot-friendly than Google/DuckDuckGo)"""
    results = []
    browser = None
    
    try:
        logger.info(f"Searching Bing for: '{query}' (page {page_num})")
        
        with sync_playwright() as p:
            browser = p.chromium.launch(
                headless=True,
                args=['--no-sandbox', '--disable-dev-shm-usage']
            )
            
            context = browser.new_context(
                user_agent=get_random_user_agent(),
                viewport={'width': 1920, 'height': 1080}
            )
            
            page = context.new_page()
            page.set_default_timeout(timeout)
            
            # Calculate first parameter for pagination
            first = (page_num - 1) * RESULTS_PER_PAGE + 1
            
            # Build Bing search URL
            params = {
                'q': query,
                'first': first if first > 1 else None,
                'count': RESULTS_PER_PAGE
            }
            
            # Remove None values
            params = {k: v for k, v in params.items() if v is not None}
            search_url = f"https://www.bing.com/search?{urlencode(params)}"
            
            logger.info(f"Bing URL: {search_url}")
            
            # Navigate to Bing
            page.goto(search_url, wait_until="domcontentloaded")
            page.wait_for_timeout(random.randint(1000, 3000))  # Random delay
            
            # Wait for results to load
            try:
                page.wait_for_selector('.b_algo', timeout=10000)
            except PlaywrightTimeout:
                logger.warning("Bing results took too long to load")
            
            # Extract results
            result_elements = page.query_selector_all('.b_algo')
            logger.info(f"Found {len(result_elements)} Bing result containers")
            
            for i, result in enumerate(result_elements):
                try:
                    # Get title and link
                    title_link = result.query_selector('h2 a')
                    if not title_link:
                        continue
                        
                    title = title_link.inner_text().strip()
                    url = title_link.get_attribute('href')
                    
                    if title and url and url.startswith(('http://', 'https://')):
                        results.append({
                            "rank": first + i,
                            "title": title[:150] + "..." if len(title) > 150 else title,
                            "url": url
                        })
                        logger.debug(f"Bing result {i+1}: {title[:50]}...")
                
                except Exception as e:
                    logger.warning(f"Error extracting Bing result {i}: {e}")
                    continue
            
            # Check for more results (Bing pagination)
            has_more = len(result_elements) >= RESULTS_PER_PAGE
            next_link = page.query_selector('a.sb_pagN')
            if next_link:
                has_more = True
            
            browser.close()
            return results, has_more
            
    except Exception as e:
        if browser:
            try:
                browser.close()
            except:
                pass
        raise SearchError(f"Bing search failed: {str(e)}")

def search_startpage(query, page_num=1, timeout=DEFAULT_TIMEOUT):
    """Search using Startpage (privacy-focused, less blocking)"""
    results = []
    browser = None
    
    try:
        logger.info(f"Searching Startpage for: '{query}' (page {page_num})")
        
        with sync_playwright() as p:
            browser = p.chromium.launch(
                headless=True,
                args=['--no-sandbox', '--disable-dev-shm-usage']
            )
            
            context = browser.new_context(
                user_agent=get_random_user_agent(),
                viewport={'width': 1920, 'height': 1080}
            )
            
            page = context.new_page()
            page.set_default_timeout(timeout)
            
            # Build Startpage search URL
            start_num = (page_num - 1) * RESULTS_PER_PAGE
            params = {
                'q': query,
                'segment': 'startpage.default'
            }
            if start_num > 0:
                params['page'] = page_num
                
            search_url = f"https://www.startpage.com/sp/search?{urlencode(params)}"
            logger.info(f"Startpage URL: {search_url}")
            
            # Navigate to Startpage
            page.goto(search_url, wait_until="domcontentloaded")
            page.wait_for_timeout(random.randint(2000, 4000))
            
            # Wait for results
            try:
                page.wait_for_selector('.w-gl__result', timeout=10000)
            except PlaywrightTimeout:
                logger.warning("Startpage results took too long to load")
            
            # Extract results
            result_elements = page.query_selector_all('.w-gl__result')
            logger.info(f"Found {len(result_elements)} Startpage results")
            
            for i, result in enumerate(result_elements):
                try:
                    title_link = result.query_selector('.w-gl__result-title a')
                    if not title_link:
                        continue
                        
                    title = title_link.inner_text().strip()
                    url = title_link.get_attribute('href')
                    
                    if title and url and url.startswith(('http://', 'https://')):
                        results.append({
                            "rank": start_num + i + 1,
                            "title": title[:150] + "..." if len(title) > 150 else title,
                            "url": url
                        })
                        logger.debug(f"Startpage result {i+1}: {title[:50]}...")
                
                except Exception as e:
                    logger.warning(f"Error extracting Startpage result {i}: {e}")
                    continue
            
            # Check for more results
            has_more = len(result_elements) >= RESULTS_PER_PAGE
            next_button = page.query_selector('a.w-gl__paging__next-button')
            if next_button:
                has_more = True
            
            browser.close()
            return results, has_more
            
    except Exception as e:
        if browser:
            try:
                browser.close()
            except:
                pass
        raise SearchError(f"Startpage search failed: {str(e)}")

def search_searx(query, page_num=1, timeout=DEFAULT_TIMEOUT):
    """Search using SearX (open source search aggregator)"""
    results = []
    browser = None
    
    # Public SearX instances (rotate between them)
    searx_instances = [
        'https://searx.be',
        'https://search.bus-hit.me',
        'https://searx.tuxcloud.net',
        'https://searx.prvcy.eu',
    ]
    
    instance = random.choice(searx_instances)
    
    try:
        logger.info(f"Searching SearX ({instance}) for: '{query}' (page {page_num})")
        
        with sync_playwright() as p:
            browser = p.chromium.launch(
                headless=True,
                args=['--no-sandbox', '--disable-dev-shm-usage']
            )
            
            context = browser.new_context(
                user_agent=get_random_user_agent(),
                viewport={'width': 1920, 'height': 1080}
            )
            
            page = context.new_page()
            page.set_default_timeout(timeout)
            
            # Build SearX search URL
            params = {
                'q': query,
                'category_general': 'on',
                'pageno': page_num
            }
            
            search_url = f"{instance}/search?{urlencode(params)}"
            logger.info(f"SearX URL: {search_url}")
            
            # Navigate to SearX
            page.goto(search_url, wait_until="domcontentloaded")
            page.wait_for_timeout(random.randint(1500, 3500))
            
            # Wait for results
            try:
                page.wait_for_selector('.result', timeout=8000)
            except PlaywrightTimeout:
                logger.warning("SearX results took too long to load")
            
            # Extract results
            result_elements = page.query_selector_all('.result')
            logger.info(f"Found {len(result_elements)} SearX results")
            
            start_rank = (page_num - 1) * RESULTS_PER_PAGE + 1
            
            for i, result in enumerate(result_elements[:RESULTS_PER_PAGE]):
                try:
                    title_link = result.query_selector('h3 a, .result-title a')
                    if not title_link:
                        continue
                        
                    title = title_link.inner_text().strip()
                    url = title_link.get_attribute('href')
                    
                    if title and url and url.startswith(('http://', 'https://')):
                        results.append({
                            "rank": start_rank + i,
                            "title": title[:150] + "..." if len(title) > 150 else title,
                            "url": url
                        })
                        logger.debug(f"SearX result {i+1}: {title[:50]}...")
                
                except Exception as e:
                    logger.warning(f"Error extracting SearX result {i}: {e}")
                    continue
            
            # Check for more results
            has_more = len(result_elements) >= RESULTS_PER_PAGE
            next_link = page.query_selector('.paging a[href*="pageno"]')
            if next_link:
                has_more = True
            
            browser.close()
            return results, has_more
            
    except Exception as e:
        if browser:
            try:
                browser.close()
            except:
                pass
        raise SearchError(f"SearX search failed: {str(e)}")

def search_web(query, page_num=1, search_engine="bing"):
    """
    Main search function that tries different search engines
    """
    engines = {
        "bing": search_bing,
        "startpage": search_startpage,
        "searx": search_searx
    }
    
    if search_engine in engines:
        return engines[search_engine](query, page_num)
    else:
        # Try engines in order until one works
        last_error = None
        for engine_name, engine_func in engines.items():
            try:
                logger.info(f"Trying {engine_name}...")
                return engine_func(query, page_num)
            except SearchError as e:
                last_error = e
                logger.warning(f"{engine_name} failed: {e}")
                continue
        
        # If all engines failed
        raise SearchError(f"All search engines failed. Last error: {last_error}")

class EnhancedSearchApp:
    def __init__(self, root):
        self.root = root
        self.setup_ui()
        self.setup_data()
        
    def setup_ui(self):
        """Initialize the user interface"""
        self.root.title("Multi-Engine Web Search")
        self.root.geometry("1100x700")
        self.root.minsize(900, 500)
        
        # Configure styles
        style = ttk.Style()
        style.configure('Title.TLabel', font=('Arial', 10, 'bold'))
        
        # Main frame
        main_frame = ttk.Frame(self.root, padding=10)
        main_frame.pack(fill="both", expand=True)
        
        # Search frame
        search_frame = ttk.LabelFrame(main_frame, text="Search Configuration", padding=10)
        search_frame.pack(fill="x", pady=(0, 10))
        
        # Search engine selection
        engine_frame = ttk.Frame(search_frame)
        engine_frame.pack(fill="x", pady=(0, 5))
        
        ttk.Label(engine_frame, text="Search Engine:").pack(side="left", padx=(0, 5))
        self.engine_var = tk.StringVar(value="auto")
        engine_combo = ttk.Combobox(engine_frame, textvariable=self.engine_var, width=15,
                                   values=["auto", "bing", "startpage", "searx"])
        engine_combo.pack(side="left", padx=(0, 20))
        
        # Query input
        query_frame = ttk.Frame(search_frame)
        query_frame.pack(fill="x", pady=(0, 5))
        
        ttk.Label(query_frame, text="Query:").pack(side="left", padx=(0, 5))
        self.query_var = tk.StringVar()
        self.entry = ttk.Entry(query_frame, textvariable=self.query_var, font=('Arial', 10))
        self.entry.pack(side="left", fill="x", expand=True, padx=(0, 5))
        self.entry.bind("<Return>", lambda e: self.search())
        self.entry.focus_set()
        
        # Buttons frame
        btn_frame = ttk.Frame(query_frame)
        btn_frame.pack(side="right")
        
        self.search_btn = ttk.Button(btn_frame, text="🔍 Search", command=self.search)
        self.search_btn.pack(side="left", padx=2)
        
        self.clear_btn = ttk.Button(btn_frame, text="🗑️ Clear", command=self.clear_results)
        self.clear_btn.pack(side="left", padx=2)
        
        # Status bar
        self.status_var = tk.StringVar(value="Ready to search")
        self.status_label = ttk.Label(search_frame, textvariable=self.status_var, 
                                     foreground="blue", font=('Arial', 9))
        self.status_label.pack(fill="x")
        
        # Results frame
        results_frame = ttk.LabelFrame(main_frame, text="Search Results", padding=5)
        results_frame.pack(fill="both", expand=True, pady=(0, 10))
        
        # Treeview with scrollbars
        tree_frame = ttk.Frame(results_frame)
        tree_frame.pack(fill="both", expand=True)
        
        columns = ("rank", "title", "url")
        self.tree = ttk.Treeview(tree_frame, columns=columns, show="headings", height=20)
        
        # Configure columns
        self.tree.heading("rank", text="#")
        self.tree.column("rank", width=50, anchor="center", minwidth=40)
        
        self.tree.heading("title", text="Title")
        self.tree.column("title", width=550, anchor="w", minwidth=200)
        
        self.tree.heading("url", text="URL")
        self.tree.column("url", width=450, anchor="w", minwidth=200)
        
        # Scrollbars
        v_scrollbar = ttk.Scrollbar(tree_frame, orient="vertical", command=self.tree.yview)
        h_scrollbar = ttk.Scrollbar(tree_frame, orient="horizontal", command=self.tree.xview)
        self.tree.configure(yscrollcommand=v_scrollbar.set, xscrollcommand=h_scrollbar.set)
        
        # Pack treeview and scrollbars
        self.tree.grid(row=0, column=0, sticky="nsew")
        v_scrollbar.grid(row=0, column=1, sticky="ns")
        h_scrollbar.grid(row=1, column=0, sticky="ew")
        
        tree_frame.grid_rowconfigure(0, weight=1)
        tree_frame.grid_columnconfigure(0, weight=1)
        
        # Bind events
        self.tree.bind("<Double-1>", self.open_selected)
        self.tree.bind("<Return>", self.open_selected)
        self.tree.bind("<Button-3>", self.show_context_menu)
        
        # Navigation frame
        nav_frame = ttk.Frame(main_frame)
        nav_frame.pack(fill="x")
        
        # Navigation buttons
        nav_buttons = ttk.Frame(nav_frame)
        nav_buttons.pack(side="left")
        
        self.prev_btn = ttk.Button(nav_buttons, text="⬅️ Previous", command=self.prev_page)
        self.prev_btn.pack(side="left", padx=2)
        
        self.next_btn = ttk.Button(nav_buttons, text="Next ➡️", command=self.next_page)
        self.next_btn.pack(side="left", padx=2)
        
        # Page info
        self.page_var = tk.StringVar(value="Page 0/0")
        self.page_label = ttk.Label(nav_frame, textvariable=self.page_var, style='Title.TLabel')
        self.page_label.pack(side="left", padx=20)
        
        # Results count and engine info
        self.info_var = tk.StringVar(value="")
        self.info_label = ttk.Label(nav_frame, textvariable=self.info_var)
        self.info_label.pack(side="right")
        
        # Context menu
        self.context_menu = tk.Menu(self.root, tearoff=0)
        self.context_menu.add_command(label="Open Link", command=self.open_selected)
        self.context_menu.add_command(label="Copy URL", command=self.copy_url)
        self.context_menu.add_command(label="Copy Title", command=self.copy_title)
        
    def setup_data(self):
        """Initialize data structures"""
        self.pages = {}
        self.has_next = {}
        self.current_page = 0
        self.query = ""
        self.search_thread = None
        self.used_engine = ""
        
    def set_status(self, text, color="blue"):
        """Update status label"""
        self.status_var.set(text)
        self.status_label.configure(foreground=color)
        self.root.update_idletasks()
        
    def clear_results(self):
        """Clear all search results"""
        for item in self.tree.get_children():
            self.tree.delete(item)
        
        self.pages.clear()
        self.has_next.clear()
        self.current_page = 0
        self.query = ""
        self.query_var.set("")
        self.used_engine = ""
        
        self.page_var.set("Page 0/0")
        self.info_var.set("")
        self.set_status("Results cleared", "green")
        self.update_navigation_buttons()
        
    def search(self):
        """Start a new search"""
        query = self.query_var.get().strip()
        if not query:
            messagebox.showwarning("Warning", "Please enter a search query.")
            self.entry.focus_set()
            return
        
        if self.search_thread and self.search_thread.is_alive():
            self.set_status("Search already in progress...", "orange")
            return
            
        self.query = query
        self.pages.clear()
        self.has_next.clear()
        self.current_page = 1
        self.load_page(1)
        
    def load_page(self, page_num):
        """Load a specific page"""
        if page_num in self.pages:
            self.show_page(page_num)
            return
            
        self.set_loading_state(True)
        engine = self.engine_var.get()
        engine_text = engine if engine != "auto" else "auto (trying multiple)"
        self.set_status(f"Searching with {engine_text}... (page {page_num})", "blue")
        
        self.search_thread = threading.Thread(
            target=self._fetch_page_thread,
            args=(page_num, engine),
            daemon=True
        )
        self.search_thread.start()
        
    def _fetch_page_thread(self, page_num, engine):
        """Background search thread"""
        try:
            if engine == "auto":
                results, has_next = search_web(self.query, page_num)
                self.used_engine = "auto"
            else:
                results, has_next = search_web(self.query, page_num, engine)
                self.used_engine = engine
                
            self.pages[page_num] = results
            self.has_next[page_num] = has_next
            
            self.root.after(0, lambda: self.show_page(page_num))
            self.root.after(0, lambda: self.set_loading_state(False))
            
        except SearchError as e:
            self.root.after(0, lambda: self.handle_search_error(str(e)))
            self.root.after(0, lambda: self.set_loading_state(False))
        except Exception as e:
            error_msg = f"Unexpected error: {str(e)}"
            logger.error(f"Search thread error: {error_msg}")
            logger.error(traceback.format_exc())
            self.root.after(0, lambda: self.handle_search_error(error_msg))
            self.root.after(0, lambda: self.set_loading_state(False))
            
    def handle_search_error(self, error_msg):
        """Handle search errors"""
        self.set_status("Search failed", "red")
        messagebox.showerror("Search Error", 
                           f"Search failed: {error_msg}\n\n"
                           "Try:\n• Different search engine\n• Check internet connection\n"
                           "• Wait a moment and try again")
            
    def show_page(self, page_num):
        """Display results for a page"""
        self.clear_tree()
        
        results = self.pages.get(page_num, [])
        if not results:
            self.set_status(f"No results found for page {page_num}", "orange")
            return
            
        # Populate treeview
        for result in results:
            self.tree.insert("", "end", values=(
                result["rank"],
                result["title"],
                result["url"]
            ))
            
        # Update UI
        self.current_page = page_num
        total_pages = self.estimate_total_pages()
        self.page_var.set(f"Page {page_num}/{total_pages}")
        
        start_rank = results[0]["rank"]
        end_rank = results[-1]["rank"]
        engine_info = f"via {self.used_engine}" if self.used_engine else ""
        self.info_var.set(f"Results {start_rank}-{end_rank} {engine_info}")
        
        self.set_status(f"Found {len(results)} results", "green")
        self.update_navigation_buttons()
        
    def estimate_total_pages(self):
        """Estimate total pages"""
        max_page = max(self.pages.keys()) if self.pages else 1
        if self.has_next.get(max_page, False):
            return f"{max_page}+"
        return str(max_page)
        
    def clear_tree(self):
        """Clear treeview"""
        for item in self.tree.get_children():
            self.tree.delete(item)
            
    def set_loading_state(self, loading):
        """Set UI loading state"""
        state = "disabled" if loading else "normal"
        self.search_btn.configure(state=state)
        self.clear_btn.configure(state=state)
        self.prev_btn.configure(state=state if not loading else "disabled")
        self.next_btn.configure(state=state if not loading else "disabled")
        
    def update_navigation_buttons(self):
        """Update navigation button states"""
        self.prev_btn.configure(state="normal" if self.current_page > 1 else "disabled")
        self.next_btn.configure(state="normal" if self.has_next.get(self.current_page, False) else "disabled")
        
    def next_page(self):
        """Go to next page"""
        if self.has_next.get(self.current_page, False):
            self.load_page(self.current_page + 1)
            
    def prev_page(self):
        """Go to previous page"""
        if self.current_page > 1:
            self.load_page(self.current_page - 1)
            
    def open_selected(self, event=None):
        """Open selected URL"""
        selection = self.tree.selection()
        if not selection:
            return
            
        item = self.tree.item(selection[0])
        url = item["values"][2]
        
        if url:
            try:
                webbrowser.open(url)
                self.set_status(f"Opened: {url[:50]}...", "green")
            except Exception as e:
                messagebox.showerror("Error", f"Failed to open URL: {str(e)}")
                
    def show_context_menu(self, event):
        """Show context menu"""
        item = self.tree.identify_row(event.y)
        if item:
            self.tree.selection_set(item)
            self.context_menu.post(event.x_root, event.y_root)
            
    def copy_url(self):
        """Copy URL to clipboard"""
        selection = self.tree.selection()
        if selection:
            url = self.tree.item(selection[0])["values"][2]
            self.root.clipboard_clear()
            self.root.clipboard_append(url)
            self.set_status("URL copied to clipboard", "green")
            
    def copy_title(self):
        """Copy title to clipboard"""
        selection = self.tree.selection()
        if selection:
            title = self.tree.item(selection[0])["values"][1]
            self.root.clipboard_clear()
            self.root.clipboard_append(title)
            self.set_status("Title copied to clipboard", "green")

def main():
    """Main application entry point"""
    try:
        root = tk.Tk()
        app = EnhancedSearchApp(root)
        
        def on_closing():
            if hasattr(app, 'search_thread') and app.search_thread and app.search_thread.is_alive():
                if messagebox.askokcancel("Quit", "Search in progress. Do you want to quit?"):
                    root.destroy()
            else:
                root.destroy()
                
        root.protocol("WM_DELETE_WINDOW", on_closing)
        
        logger.info("Multi-Engine Search Application started")
        root.mainloop()
        
    except Exception as e:
        error_msg = f"Failed to start application: {str(e)}"
        logger.error(error_msg)
        logger.error(traceback.format_exc())
        print(f"FATAL ERROR: {error_msg}")
        sys.exit(1)

if __name__ == "__main__":
    main()
