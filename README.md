# venkat.github.io

This is the source code for my personal blog, hosted at [venkat.io](https://venkat.io). It is built using [Jekyll](https://jekyllrb.com/), a static site generator.

## Getting Started

These instructions will get you a copy of the project up and running on your local machine for development and testing purposes.

### Prerequisites

You will need to have [Ruby](https://www.ruby-lang.org/en/documentation/installation/) and [Bundler](https://bundler.io/) installed on your machine.

### Installation

1.  Clone the repository:
    ```bash
    git clone https://github.com/venkat/venkat.github.io.git
    ```
2.  Navigate to the project directory:
    ```bash
    cd venkat.github.io
    ```
3.  Install the dependencies:
    ```bash
    bundle install
    ```
4.  Run the Jekyll server:
    ```bash
    bundle exec jekyll serve --watch
    ```

Your site will be available at `http://localhost:4000`.

## Directory Structure

*   `_config.yml`: Contains Jekyll configuration settings.
*   `_posts/`: Contains all the blog posts as markdown files.
*   `_layouts/`: Contains the HTML layouts for pages and posts.
*   `_drafts/`: Contains unpublished drafts.
*   `css/`: Contains the stylesheets for the site.
*   `images/`: Contains images used in the posts.
*   `documents/`: Contains documents referenced in the posts.
*   `photos/`: Contains photos used in the posts.
*   `tarballs/`: Contains compressed archives.

## Deployment

This site is automatically deployed to [GitHub Pages](https://pages.github.com/) when changes are pushed to the `master` branch. The `.github/workflows/jekyll-gh-pages.yml` file contains the GitHub Actions workflow for the deployment.

## Contributing

This is a personal blog, but if you find any issues or have suggestions, feel free to open an issue or a pull request.