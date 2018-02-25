---
path: /test/2
title: Markdown Test 2
author: Hyeseong Kim
date: 2018-02-02
tags: 
  - "test"
---

### Headers

Markdown supports two styles of headers, [Setext] [1] and [atx] [2].

Optionally, you may "close" atx-style headers. This is purely
cosmetic -- you can use this if you think it looks better. The
closing hashes don't even need to match the number of hashes
used to open the header. (The number of opening hashes
determines the header level.)

Code Highlight Test
```tsx
import * as React from 'react'
import styled from 'styled-components'
import Link from 'gatsby-link'

import theme from 'utils/theme'

interface HeaderProps {
    title: string
    fixed?: boolean
}

interface HeaderState {
    hide: boolean,
    pageYOffset: number
}

export default class Header extends React.PureComponent<HeaderProps, HeaderState> {
    static defaultProps: Partial<HeaderProps> = {
        fixed: false,
    }

    state = {
        hide: false,
        pageYOffset: 0
    }
    
    handleScroll = () => {
        const { pageYOffset } = window
        const deltaY = pageYOffset - this.state.pageYOffset
        const hide = pageYOffset !== 0 && deltaY >= 0

        this.setState({ hide, pageYOffset })
    }

    componentDidMount() {
        if (!this.props.fixed) {
            window.addEventListener('scroll', this.handleScroll)
        }
    }

    componentWillUnmount() {
        if (!this.props.fixed){
            window.removeEventListener('scroll', this.handleScroll)
        }
    }

    render() {
        return (
            <Container className={this.state.hide ? 'hide' : ''}>
                <HomeLink to='/'>{this.props.title}</HomeLink>
            </Container>
        )
    }
}

const Container = styled.header`
    display: flex;
    align-items: center;
    margin: 0;
    position: fixed;
    width: 100%;
    height: ${theme.headerHeight};
    background: #fff;
    border-bottom: 1px solid ${theme.grayColor};
    transition: transform .5s ease;
    z-index: 2;
    
    &.hide {
        transform: translateY(-${theme.headerHeight});
    }
`

const HomeLink = styled(Link) `
    position: absolute;
    left: 20px;
    text-decoration: none;
    font-size: 24px;
    font-weight: bold;
    color: black;
`
```
